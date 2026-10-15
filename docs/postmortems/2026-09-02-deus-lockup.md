# Postmortem: deus silent lockup and total tailnet plane outage

**Date of incident:** 2026-09-02, 23:12 to 23:33 SGT
**Duration:** ~21 minutes hard down; full recovery required physical intervention
**Author:** Barney
**Status:** Root cause narrowed to hardware/firmware; remediation in progress
**Severity:** High. Complete loss of all administrative access, including the tooling
required to diagnose the incident

---

## 1. Summary

At approximately 23:12 SGT on 2026-09-02, `deus` (Lenovo M910q, i7-6700T, 192.168.1.250)
stopped executing. There was no shutdown sequence, no kernel panic, no OOM kill, and no
final journal entry beyond a routine `tailscaled` keepalive. The machine was idle at the
time: 17% memory used, load below 1.0, run queue empty.

Because `deus` is the sole Tailscale subnet router advertising `192.168.1.0/24`, its
death removed the only network path from outside the LAN into the cluster. Every other
node remained healthy and continued serving public traffic through the Cloudflare tunnel,
but became unreachable for administration.

Recovery required physically pressing the power button. The cluster self-healed on boot.

This was **not** the first occurrence. Journal analysis found three silent lockups in four
months, all with the same signature.

---

## 2. Impact

**Lost:**

- SSH to every node (all reached via the subnet route)
- `kubectl` via the VIP at 192.168.1.241
- Every `*.lab.packetcraft.dev` service: ArgoCD, Grafana, Headlamp, Seafile, code-server,
  site-dev, sim-next
- All observability, since Grafana and Headlamp are themselves on the lab plane

**Unaffected:**

- `site.packetcraft.dev` and its Go backend
- `site-admin.packetcraft.dev`
- The cluster itself: etcd retained quorum on `opus` and `sol`, cloudflared connectors
  and Traefik stayed up on other nodes, no workload data was lost

The public tunnel path survived because both `cloudflared` (2 replicas) and Traefik
(2 replicas, host-topology anti-affinity) are unpinned and were scheduled elsewhere. That
resilience was by design and it worked.

---

## 3. Timeline (SGT)

| Time | Event |
|---|---|
| Sep 1, 22:24:39 | Unattended apt transaction: `tailscale` 1.102.2 to 1.102.3, `linux-firmware`, `network-manager`, `snapd`, `util-linux` set |
| Sep 1, 22:31:16 | Intentional reboot. Clean, the journal ends with `Journal stopped` |
| Sep 1, 22:33:14 | Boot `-1` begins. Normal operation for ~24.7 hours |
| Sep 2, 23:10:09 | Final `sysstat-collect` sample. Memory 17.28%, load 0.72, runq 0, 1 blocked |
| Sep 2, 23:11:31 | Last k3s entry: routine etcd `apply request took too long` (134ms vs 100ms expected) on a Longhorn node read |
| Sep 2, 23:12:34 | Last journal entry of any kind: `tailscaled` disco keepalive to a peer |
| Sep 2, 23:12 to 23:20 | Death window. The 23:20:09 sysstat sample never ran |
| Sep 2, ~23:15 | Operator notices `kubectl` timeouts against the VIP |
| Sep 2, 23:33:25 | Manual power-button reboot. Boot `0` begins |
| Sep 3, ~00:00 | Cluster fully reconverged |

---

## 4. Evidence

### 4.1 The machine did not shut down

Clean shutdowns on this host end with `systemd-journald: Journal stopped`. Boot `-1` ends
mid-stream on an unrelated `tailscaled` line. No `Stopping`, no `Reached target Shutdown`,
nothing.

### 4.2 It was idle, not stressed

`sar` data survived on disk independently of journald:

```
                kbmemfree   kbavail  %memused  %commit  ldavg-1  runq-sz  blocked
11:00:09 PM      9078352  12858816     17.77    44.21     1.09        0        0
11:10:09 PM      9137396  12938644     17.28    44.21     0.72        0        1
```

`%commit` is pinned at 44.21 across every sample of the evening, so no leak and no growth. The
sample 2.5 minutes before death is indistinguishable from one taken an hour earlier.

### 4.3 No crash artifacts

- `journalctl -b -1 -k | grep -iE "oom|mce|panic|soft lockup|hung task|EDAC"` returns nothing
  but boot-time registration messages
- `/sys/fs/pstore/` is empty and mounted; no firmware panic capture
- `/var/crash/` is empty
- No kubelet eviction, no `MemoryPressure`, no `DiskPressure`

### 4.4 The NMI watchdog did not fire

`NMI watchdog: Enabled` at boot. It catches CPUs spinning with interrupts disabled and
dumps a backtrace. Its silence suggests the CPU **stopped executing** rather than got
stuck in a bad loop.

### 4.5 This has happened before

Three of twenty boots ended without `Journal stopped`:

| Boot | Ended | Last line | Uptime |
|---|---|---|---|
| -18 | 2026-05-13 00:50:03 | `sysstat-collect` | ~2.5 h |
| -3 | 2026-07-19 19:10:07 | `sysstat-collect` | ~2.8 h |
| -1 | 2026-09-02 23:12:34 | `tailscaled` keepalive | ~24.7 h |

The other seventeen closed cleanly. Same signature each time: idle machine, mid-stream
journal truncation, no diagnostic residue.

---

## 5. Root cause

**Immediate cause:** an unrecoverable hardware or firmware lockup on `deus` with the
machine at idle. The CPU ceased executing between one instruction and the next; no fault
handler ran and no buffered log data reached disk.

**Root cause of the outage** (as distinct from the hang): `deus` is the only node
advertising the `192.168.1.0/24` subnet route. All administrative access (SSH, kubectl,
and every `*.lab` hostname) depends on that single advertisement. One host failure
removed the entire remote-access and observability plane.

### Ranked hypotheses for the hang

1. **Deep C-state entry failure.** All three hangs caught the machine idle. An idle
   Skylake dropping into C6/C8 and failing to wake produces exactly this signature: total
   silence, no watchdog, no logs. Consistent with the NMI watchdog never firing.
2. **Memory corruption.** `EDAC ie31200: No ECC support`, so this host cannot detect a bit
   flip. A flip in a page table or IDT entry can wedge the machine hard enough that no
   handler runs. The failure would be undetectable by design, so absence of evidence is
   not evidence of absence. Argues against: corruption more typically produces oopses or
   panics, which would have logged something.
3. **Power delivery.** External 65W bricks on these minis degrade and cause silent idle
   lockups with no logs. Cheap to test by substitution.

**Ruled out:** resource exhaustion (sar), OOM (no `node_vmstat_oom_kill`, no dmesg),
kernel panic (empty pstore), microcode (`0x000000f0` is current for Skylake), and the
Sep 1 firmware upgrade as sole cause (two prior hangs predate it, though it may have
influenced the 24.7h interval).

---

## 6. What went wrong in the response

Worth recording, because the diagnostic path was longer than it needed to be.

- **A false observation drove an early hypothesis.** An impression that Prometheus kept
  scraping `deus` through the outage led to a "the host stayed up, only Tailscale died"
  reframe. That is incompatible with a hard lockup and was almost certainly a stale
  Grafana panel. *Still to verify:* `up{instance=~"deus.*"}` should show a gap from ~23:12
  to 23:33.
- **A date misread cost a cycle.** A burst of seven veth teardowns was read as immediately
  pre-crash; they were dated Sep 1, a day earlier, and were routine pod churn.
- **Diagnosis happened after a reboot.** The reboot was necessary for recovery but
  destroyed the live state. Persistent journald saved the investigation.
- **No alerting fired.** The outage was discovered by a human running `kubectl`.
  Alertmanager could not have notified anyway, because it is behind the failed plane.

---

## 7. Action items

### Immediate (same day)

| # | Action | Rationale |
|---|---|---|
| 1 | `RuntimeWatchdogSec=60` in `/etc/systemd/system.conf`, `iTCO_wdt` loaded, on **all five nodes** | Converts a 21-minute outage requiring physical presence into a 60-second reboot |
| 2 | `intel_idle.max_cstate=1` on deus's kernel cmdline | Directly tests hypothesis 1. Costs a few watts of idle draw |
| 3 | `tailscale set --accept-routes=false` on `opus` and `sol` | See section 9, an active fault |
| 4 | Second subnet router advertising `192.168.1.0/24` | Removes the SPOF that turned a node failure into a total outage |

### Short term (this week)

| # | Action | Rationale |
|---|---|---|
| 5 | Codify 1 to 4 in the Ansible `base` role | Hand-applied fixes vanish on rebuild |
| 6 | `netconsole` shipping kernel messages to another node over UDP | Captures the final moments journald can never flush |
| 7 | memtest86+ overnight on deus | Tests hypothesis 2. Requires a maintenance window |
| 8 | Swap deus's power brick | Tests hypothesis 3. Cheap if a spare 65W exists |
| 9 | Remove the `cloudflared` host package (`apt purge`) | Systemd unit already removed, but apt keeps upgrading a binary that should not exist on the host |
| 10 | Verify `up{instance=~"deus.*"}` gap | Closes the discrepancy in section 6 |

### Medium term

| # | Action | Rationale |
|---|---|---|
| 11 | Move `*.lab` DNS off subnet-routed addresses (section 10) | Eliminates the dependency rather than making it redundant |
| 12 | An alert path that survives losing the lab plane | Alertmanager currently cannot report its own plane's failure |
| 13 | Track deus uptime; if it passes 30 days on `max_cstate=1`, close hypothesis 1 | Prior intervals were 2.5h, 2.8h, 24.7h, so a month of clean uptime is meaningful |

---

## 8. Findings for the running lessons log

- **A clean shutdown ends with `systemd-journald: Journal stopped`.** Its absence at the
  end of a boot is a reliable marker of a hard hang. `journalctl --list-boots` plus a
  last-line check per boot turns an incident into a pattern.
- **`sar` survives what journald loses.** sysstat's 10-minute on-disk samples persisted
  through a hang that flushed nothing, and were decisive in ruling out resource
  exhaustion. Keep sysstat installed on every node.
- **A silent NMI watchdog is diagnostic.** It firing means a CPU spun; it staying quiet
  during a total freeze means the CPU stopped. That distinction narrows the suspect list.
- **No ECC means a whole failure class is undetectable, not absent.** `EDAC ie31200: No
  ECC support` should be read as "this machine cannot tell you about memory faults."
- **Diagnostic tooling must not live behind the thing it diagnoses.** Every admin surface
  was on the lab plane; the outage blinded the operator to itself.

---

## 9. Follow-on incident: accept-routes on opus and sol

While remediating, Tailscale was installed on `opus` and `sol` to give them independent
tailnet identities. This introduced a second, distinct fault.

**Symptom:** intermittent `kubectl` failures against the VIP, with some calls timing out,
others returning normally within seconds of each other. Nodes flapping between `Ready` and
`NotReady`.

**Cause:** both nodes came up with `--accept-routes` enabled and installed deus's
`192.168.1.0/24` advertisement into Tailscale's policy routing table:

```
192.168.1.0/24 dev tailscale0 table 52
192.168.1.0/24 dev eno1 proto kernel scope link src 192.168.1.252
```

Tailscale's ip rules sit near priority 5270; the main table sits at 32766. Table 52 wins.
Both nodes were routing their **own LAN traffic** (etcd peer traffic, kubelet-to-apiserver
calls, kube-vip leader election) into the tunnel, across to deus, and back out onto the
same physical wire.

**Confirming detail:** `opus` held the VIP and stayed `Ready`, because the `local` table
(priority 0) beats table 52 and it reached the apiserver via itself. `sol` had to cross
the LAN and went `NotReady`. That asymmetry is the fingerprint of this fault.

**Fix:** `sudo tailscale set --accept-routes=false` on both. Use `set`, not `up`, or the
rest of the flags reset.

**Note:** `tailscale status --json | jq '.Self.RouteAll'` returns `null` regardless of the
setting, because `.Self` is a PeerStatus object with no such field. Check `ip route show table 52`
or `tailscale debug prefs` instead.

---

## 10. Recommendation: removing the subnet-router single point of failure

The subnet router is load-bearing for everything administrative and has no redundancy.
Four layers of fix, in increasing order of thoroughness. Layer 1 is required regardless.

### Layer 1: correct the current state (do first)

1. Disable `--accept-routes` on `opus` and `sol`. Verify `ip route show table 52` no
   longer contains `192.168.1.0/24`.
2. Confirm `deus` itself does not accept routes, since a subnet router must never accept its
   own advertisement back.
3. Pin both settings in the Ansible Tailscale role so a rebuild cannot reintroduce them.

Non-negotiable rule for the role: **cluster nodes get tailnet identities, not accepted
routes.**

### Layer 2: HA subnet routing (removes the SPOF)

Tailscale supports advertising the same prefix from multiple nodes. It elects one primary
and fails over automatically if that node goes offline.

On `opus` and `sol`:

```bash
# forwarding must be on, or an elected primary blackholes the prefix
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding=1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

sudo tailscale set --advertise-routes=192.168.1.0/24
# then approve the routes in the admin console
```

Verify which node currently owns the prefix:

```bash
tailscale status --json | jq '.Peer[] | select(.PrimaryRoutes) | {HostName, PrimaryRoutes}'
```

**Caveats.** Only one router is primary at a time. This is failover, not load balancing.
Failover is a control-plane decision and takes tens of seconds to a couple of minutes, so
expect a brief gap rather than seamless continuity. And `ip_forward` **must** be set on
every candidate before advertising; an elected primary without it silently blackholes the
entire prefix, which is a worse failure than the one being fixed.

This is the minimum viable fix and it is what makes `kubectl` against the VIP survive a
node loss.

### Layer 3: remove the dependency for services (the real fix)

Layer 2 makes the subnet route redundant. Layer 3 stops most traffic needing it at all.

Today, `*.lab.packetcraft.dev` resolves to a private `192.168.1.x` address, so every lab
request traverses a subnet router. It does not have to. Point those records at nodes'
`100.x` tailnet addresses instead, and traffic then flows node-to-node over WireGuard with no
router in the path.

Mechanically: Traefik listens on host ports via ServiceLB, so a node's tailnet IP on
`:80`/`:443` reaches Traefik the same way its LAN IP does. Publishing A records for two or
three nodes' tailnet IPs gives DNS-level redundancy on top.

Sequence:

1. Collect tailnet IPs for the nodes running Traefik (`tailscale ip -4`).
2. Confirm Traefik answers on `100.x:80` from another tailnet device before changing DNS.
3. Update the grey-cloud A records in Cloudflare, one hostname at a time, starting with
   something low-stakes like `site-dev`.
4. Verify, then migrate the rest. Keep Headlamp and Grafana for last, since they are the
   diagnostic surfaces.

**What Layer 3 cannot cover.** The apiserver VIP at `192.168.1.241` is a LAN address by
construction and still needs the subnet route. Two options: keep Layer 2 for kubectl
(simplest), or add each node's tailnet IP to the k3s `--tls-san` list and point kubeconfig
at a node directly, which loses VIP failover in exchange for removing the route
dependency. Layer 2 is the better trade for now.

### Layer 4: Tailscale Kubernetes Operator (future)

Already on the roadmap for the penny-dev LAN exposure problem. It would let services be
exposed as first-class tailnet devices with their own identities and ACLs, removing both
the subnet-route dependency and the LAN port exposure in one move. Worth evaluating after
Layers 1 to 3 are stable. It changes the ingress model and should not be attempted during
incident remediation.

### Recommended order

1. Layer 1 today, because it is an active fault, not a hardening exercise.
2. Watchdog on all nodes (action item 1), the cheapest failure-mode improvement available.
3. Layer 2 before the memtest window, so losing `deus` deliberately does not repeat this
   outage.
4. Layer 3 incrementally, low-stakes hostnames first.
5. Layer 4 evaluated separately, on its own merits.
