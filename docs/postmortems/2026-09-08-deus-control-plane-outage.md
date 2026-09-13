# Post-mortem: deus control-plane outage, 2026-09-08 to 2026-09-12

**Incident start:** 2026-09-08, 21:47 SGT
**Resolved:** 2026-09-12, ~13:15 SGT
**Duration:** ~3 days 15 hours
**Author:** Barney
**Status:** Root cause confirmed. Remediation applied. Follow-up actions open.
**Severity:** SEV-1. Full control-plane loss, total loss of remote access, cluster
running without etcd fault tolerance for the entire window.

---

## 1. Summary

Tailscale on `deus` was configured to accept advertised routes while
simultaneously acting as the subnet router advertising `192.168.1.0/24`. A
kube-vip change, deployed minutes earlier, altered the address configuration on
`eno1`. Interface address changes cause `tailscaled` to run a full `Reconfig`,
which reinstalled the LAN prefix into routing table 52 via `tailscale0`.

From that point, `deus` could originate traffic normally but could not answer
it. Inbound LAN packets reached the interface and were visible to `tcpdump`, but
the return path for their source addresses resolved to the tunnel rather than
`eno1`, so nothing was answered. SSH, ICMP, the Kubernetes API, kubelet
registration and etcd peer traffic all failed in the same way and for the same
reason.

The resulting etcd quorum loss terminated `k3s.service`. Because K3s runs the
control plane as a single binary, the loss of one component's lease took down
the apiserver, scheduler, controller-manager, kubelet and embedded etcd
together.

The condition survived multiple power cycles because Tailscale preferences are
persistent state, not runtime state. This was the strongest available diagnostic
signal and it was not acted on until later.

A second, independent fault was discovered during remediation: `ufw` was active
on `deus` despite the intended baseline being disabled, and was blocking the
etcd peer port from both peers.

**Root cause:** `tailscale --accept-routes` enabled on the node that advertises
the LAN prefix, made live by a kube-vip-driven interface change.

**Underlying cause:** Tailscale configuration on this cluster is applied by hand
per node and is not managed by Ansible, so `deus` had diverged from `opus` and
`sol` with no mechanism to detect it.

---

## 2. Impact

| Area | Impact |
|---|---|
| `deus` | Unreachable inbound for ~3 days 15 hours. SSH, ICMP, API, kubelet all failed. |
| Kubernetes API | `kubectl get nodes` reported `deus` NotReady; API calls that needed to reach `deus` failed. |
| etcd | Ran at 2 of 3 members for the entire window. Quorum held, but with zero fault tolerance. A single further member failure would have meant a full cluster outage and a restore from snapshot. |
| Remote access | Total, for the whole cluster. `deus` is the only Tailscale subnet router, so its loss removed remote access to the two healthy nodes as well. |
| Monitoring | Continuous Prometheus scrape gap for `deus`. |
| Control plane HA | kube-vip withdrew `192.168.1.241/32`. |
| Data | None. No data loss. |

The etcd exposure is the most serious line in this table. The cluster spent the
better part of four days one node away from a restore-from-snapshot event, and
nothing alerted on that state.

---

## 3. Timeline

### Incident, 2026-09-08 (SGT)

| Time | Event |
|---|---|
| 21:35:23 | kube-vip change pushed. `kube-vip-services` pod crashlooping on `InvalidImageName: ghcr.io/kube-vip/kube-vip:<match-static-pod>`, an unsubstituted placeholder committed to the manifest and synced by ArgoCD as Healthy. |
| 21:38:01 | Placeholder corrected; `kube-vip-services-jfg7q` starts. VIP allocation for `192.168.1.242` (Traefik ingress) proceeds. |
| 21:45:41 | Last clean etcd compaction. db 31 MB, revision 5701562. Cluster fully healthy. |
| 21:47:38 | `tailscaled` runs a full `Reconfig` in response to interface address change: "configuring router", "configuring DNS", `RTM_NEWROUTE: dst=192.168.1.0/24, outif=3 (tailscale0), table=52`. |
| 21:47:38.523 | First etcd `ReadIndex` request that never completes. |
| 21:47:43.52 | apiserver range reads at 4.998s and timing out. Lease renewals failing for `k3s-cloud-controller-manager`, `kube-controller-manager`, `kube-scheduler`, `cnpg-system/db9c8771.cnpg.io`. |
| 21:47:46.673 | `e116943b377acaa8 stepped down to follower since quorum is not active`. `deus` had been raft leader at term 81. |
| 21:47:48.562 | `controllermanager.go:368 "leaderelection lost"`. |
| 21:47:48 | `k3s.service: Main process exited, code=exited, status=1/FAILURE`. 17 `containerd-shim` processes orphaned. |
| 21:47:48 | kube-vip withdraws `192.168.1.241/32` from `eno1`. `tailscaled` logs a second `LinkChange` and rebinds. |
| 21:47:53 | systemd restart, counter at 1. New k3s process PID 2790593. |
| 21:48 onward | New etcd cannot rejoin. `prober detected unhealthy status ... error: EOF` against both peers, `failed to publish local member to cluster through raft` every 15s, endless `MsgPreVote` at term 81 receiving only self-votes. **This retry loop is the CPU spike seen in Grafana.** |
| Not logged | Operator notices `deus` missing in Grafana. `kubectl get nodes` shows NotReady. SSH and ping to `deus` both fail. |
| Not logged | **Power cycle attempted. SSH and ping still fail.** |

### Remediation, 2026-09-12 (Saturday)

| Time | Event |
|---|---|
| 11:31:04 | Node powered off and rebooted. Still unreachable. |
| Not logged | Display and keyboard attached directly for console access. |
| Not logged | Hardware validation run: memory test, storage, `sensors`. **All clean.** No thermal, memory or disk fault. |
| Not logged | Confirmed from the console that `deus` can ping the internet **and other LAN nodes**, but nothing can reach `deus`. Network layer functional; the node is not answering. |
| 12:26:29 | `eno1 entered promiscuous mode` as `tcpdump` starts. |
| Not logged | Ping issued from another host. **Packet observed arriving at `deus`.** No reply generated. Fault is at the response path, not at delivery. |
| Not logged | Packet inspection shows the inbound traffic arriving via Tailscale rather than as plain LAN traffic on `eno1`. |
| 12:27:02 | `eno1 left promiscuous mode` as the capture ends. |
| 12:43:08 | `ufw` found to be **active**, contrary to the intended baseline of disabled. First of 84 `[UFW BLOCK]` entries appears in `dmesg`, all TCP SYN to `DPT=2380` from `192.168.1.251` and `192.168.1.252`: the etcd peer port, blocked from both peers. |
| Not logged | Tailscale stopped and `ufw` disabled. Connectivity restored immediately. |
| 13:07:49 | Last `[UFW BLOCK]` entry. |
| ~13:15 | `deus` rejoined K3s. Tailscale re-enabled with `--accept-routes=false`, still advertising `192.168.1.0/24` as subnet router. Stable. |

---

## 4. Mechanism

### Why the node stopped answering

`deus` advertises `192.168.1.0/24` to the tailnet as a subnet router. With
`--accept-routes` also enabled, it accepts that same prefix back and installs it
into table 52 pointing at `tailscale0`.

The consequence is asymmetric:

- **Outbound to the internet** follows the default route on `eno1`. Unaffected,
  which is why internet pings from the console worked.
- **Outbound to LAN hosts** resolves via the tunnel, hairpins back out through
  the node's own subnet-router path, and still reaches its destination. Slower
  and stranger, but functional, which is why LAN pings from `deus` also worked
  and made the network look healthy from the console.
- **Inbound from LAN hosts** arrives on the wire and is captured by `tcpdump`,
  which taps below the routing and filtering layers. The reply, however, is
  routed by destination address, and that lookup now resolves to `tailscale0`.
  The response goes into the tunnel instead of back out `eno1` and never reaches
  the original sender.

This is why every inbound protocol failed identically and simultaneously: SSH,
ICMP, the API on 6443, kubelet on 10250 and etcd peer traffic on 2380 are not
five separate faults, they are one routing fault observed five times.

It also explains the etcd log signature precisely. `deus` lost both peers in the
same second because the fault is local to `deus`, not distributed across two
independent machines.

### The CPU spike was etcd, and it was a symptom

The sharp CPU rise visible in Grafana immediately before the scrape gap was the
isolated etcd member trying to rejoin. With peer connectivity gone, it runs a
tight loop of pre-vote campaigns at term 81, peer health probes against both
unreachable members across two round-trippers, and `failed to publish local
member to cluster through raft` retries every 15 seconds, while the apiserver
piles up requests against a datastore that never answers. None of that work can
succeed, and none of it backs off meaningfully.

The ordering matters: the spike **begins after** connectivity is already lost,
at 21:47:38 onward. During triage it was initially read as a possible cause,
which sent the investigation toward thermal throttling, storage stalls, memory
exhaustion and runaway workloads. All of those were later eliminated by the
console hardware tests. A CPU spike on a control-plane node that coincides with
a scrape gap should be read as a control plane failing loudly, not as a machine
under load.

### Why tcpdump was the turning point

Seeing the packet arrive eliminated an entire class of hypotheses at once:
cabling, switch port, NIC failure, upstream routing, and anything that would
have prevented delivery. What remained was necessarily a decision the node was
making about a packet it had already received. That is a routing or filtering
question, and it is the point at which the investigation narrowed to `ip route`
and `ip rule`.

### Why the power cycle did not help

Tailscale preferences live in `/var/lib/tailscale/tailscaled.state` and are
reapplied on boot. `ufw` rules are likewise persistent. A fault that survives a
power cycle cannot be transient runtime state, and that observation was
available on day one. Acting on it would have redirected the investigation from
hardware to persistent configuration roughly three days earlier.

### What kube-vip actually did

kube-vip did not misconfigure anything itself. It changed interface addressing
on `eno1` by allocating `192.168.1.242` and, later, withdrawing
`192.168.1.241/32`. Interface address changes are what trigger a `tailscaled`
`LinkChange` and full `Reconfig`, which is when the route installation happened.

The distinction matters for the fix. kube-vip was the trigger for a
misconfiguration that was already present and latent. Had `--accept-routes` not
been enabled on `deus`, the same kube-vip change would have been a non-event.
Any other cause of an interface change (DHCP lease renewal, a second VIP, a NIC
reset, an Ansible run touching networking) would have set off the same failure
eventually.

---

## 5. Second fault: ufw active contrary to baseline

`ufw` was running on `deus` with a rule set that did not permit the etcd peer
port, blocking inbound TCP 2380 from both `sol` and `opus`. The intended
baseline for cluster nodes is `ufw` disabled.

This is a genuine second fault, not a symptom of the first. It would have kept
etcd broken even after the Tailscale routing was corrected.

Two caveats on attribution, recorded for honesty:

- Tailscale and `ufw` were disabled in the same step, so their individual
  contributions were not isolated. Re-enabling Tailscale with
  `--accept-routes=false` and finding the node healthy does establish that the
  route setting was necessary to the fix.
- `ufw` alone does not explain the original symptoms. Its default `before.rules`
  permit ICMP echo requests, so a bare `ufw enable` with no rules would have
  blocked SSH but left ping working. Ping failed, so `ufw` cannot account for
  the full picture on 2026-09-08.

It remains unknown when and why `ufw` became active on `deus`, and that is an
open item.

---

## 6. Contributing factors

**Tailscale configuration is not managed by Ansible.** This is the underlying
cause and the most important line in this document. Every node's tailnet
configuration was applied by hand, so `deus` diverged from `opus` and `sol` with
no drift detection and no record of when or why. The same applies to `ufw`,
which explains how it came to be running against the intended baseline. Any
configuration that is not in the repo is a configuration nobody knows the
current value of.

**`deus` is a five-way single point of failure.** Control-plane node, etcd
member, schedulable worker, kube-vip VIP holder, and sole Tailscale subnet
router. The last role is what turned a single-node fault into a total loss of
remote access, including to the two healthy nodes that could have been used to
investigate.

**No out-of-band access.** With remote access gone and no console, IPMI, serial
or netconsole, the investigation could not begin until someone was physically in
front of the machine. This is the single largest contributor to the ~3.5 day
duration. The actual debugging, once a display was attached, took under an hour.

**etcd was not monitored.** kube-prometheus-stack's `kubeEtcd` job selects pods
labeled `component=etcd` in `kube-system`, which do not exist under K3s where
etcd is compiled into the k3s binary. The job had zero targets.
`etcd_server_has_leader == 0` and `etcd_network_peer_sent_failures_total` were
never collected, so the most diagnostic signals available were absent, and the
four-day degraded-quorum state went unalerted.

**systemd unit state was not monitored.** node-exporter runs without
`--collector.systemd`, so `k3s.service` exiting and restarting produced no
metric.

**Nothing alerted.** Detection was a human noticing a gap in a Grafana
dashboard.

**ArgoCD reported a broken manifest as Healthy.** The image tag
`ghcr.io/kube-vip/kube-vip:<match-static-pod>` synced green because the schema
was valid. Third recorded instance of this class in this cluster.

---

## 7. What went well

- etcd quorum held on `opus` and `sol`. No data loss, no restore required.
- Hardware was validated early during the console session and cleanly
  eliminated, which prevented a costly detour into replacing parts.
- `tcpdump` was reached for at the right moment and was decisive.
- The fix chosen preserves the desired functionality: `deus` still advertises
  the subnet, it simply no longer accepts it back. No capability was traded away
  for stability.
- Journald retention was sufficient to reconstruct the incident at sub-second
  resolution four days later.

---

## 8. Action items

| # | Action | Rationale | Priority |
|---|---|---|---|
| 1 | Bring Tailscale configuration under Ansible: `--accept-routes=false` on all nodes, `--advertise-routes` only where intended, applied idempotently | The underlying cause. Hand-applied config is undetectable drift | P0 |
| 2 | Bring `ufw` state under Ansible, explicitly asserting the intended baseline on every run | A firewall was running that nobody intended; assert the state rather than assume it | P0 |
| 3 | Add a second Tailscale subnet router on `sol` | Removes the amplification that made a one-node fault a whole-cluster access outage | P0 |
| 4 | Enable `etcd-expose-metrics: true` and deploy the etcd ScrapeConfig and PrometheusRule | Four days at 2 of 3 members went unalerted. `EtcdQuorumLost` and `EtcdMembersDegraded` cover exactly this | P0 |
| 5 | Add `--collector.systemd` to node-exporter, alert on `k3s.service` restarts | One line of YAML, catches the failure independently of etcd metrics | P1 |
| 6 | Configure netconsole from `deus` to `nebula`, plus `kernel.panic_on_oops=1` | Out-of-band visibility. Would not have fixed this incident, but shortens the next one | P1 |
| 7 | Add a post-change verification step for any networking, Tailscale or firewall change on a server node: `nc -vz <peer> 2380` and `etcdctl endpoint health --cluster` before walking away | The change that triggered this was pushed and not verified | P1 |
| 8 | Determine when and why `ufw` became active on `deus` | Unknown provenance of a running firewall on a cluster node | P1 |
| 9 | Implement the app-of-apps watching `argocd/` | Removes the manual-apply drift that complicates every "what changed" question | P2 |
| 10 | Taint `deus` to keep general workloads off it | Reduces blast radius on the node carrying the most cluster-critical roles | P2 |

---

## 9. Lessons

**A fault that survives a power cycle is persistent configuration.** This was
known on day one and not acted on. Runtime state, thermal conditions, memory
corruption and transient routing all clear on reboot. Anything that does not is
written down somewhere on disk, and the investigation should start from "what
config is applied at boot" rather than from hardware.

**`tcpdump` seeing a packet does not mean the host will answer it.** Capture
taps below routing and filtering. "The packet arrived" eliminates delivery
problems and narrows the question to what the kernel decides to do next, which
is a different and much smaller search space.

**Asymmetric reachability is a return-path problem.** "Can ping out, cannot be
pinged" is not a partial failure or a hang. It is a statement about where
replies are being sent, and it points directly at `ip route` and `ip rule`.

**A control plane burning CPU is usually failing, not working.** The spike was
etcd retrying, not load causing a failure. On a control-plane node, a CPU rise
that coincides with a monitoring gap should be read as the symptom it is, and
the first question should be what the process is retrying rather than what is
consuming the machine.

**A subnet router must never accept routes for the prefix it advertises.** The
node becomes reachable through a tunnel that exists only because the node is
reachable. The functionality is fully preserved by advertising without
accepting.

**Configuration outside the repo is configuration with an unknown value.** Two
separate faults in this incident, the Tailscale flag and the running firewall,
both existed because they were applied by hand on one node. Neither was
detectable before it caused an outage.

**Monitoring defaults do not survive a change of distribution.** The etcd blind
spot exists because upstream kube-prometheus-stack assumes a pod-based etcd. An
alert rule that syncs green is not an alert rule that works; validate by forcing
the condition.

**A single-binary control plane converts component failures into node
failures.** Losing one lease terminated everything. This is a deliberate K3s
tradeoff, but it means control-plane network faults have a much lower tolerance
here than on a conventional cluster.

---

## 10. Open questions

- When and by what mechanism did `ufw` become active on `deus`? Check
  `/var/log/ufw.log` for 2026-09-08 entries and the mtime on
  `/etc/ufw/user.rules` and `/etc/ufw/ufw.conf`.
- Was `--accept-routes` deliberately enabled on `deus` at some point, or set as
  a side effect of a `tailscale up` invocation that omitted flags? `tailscale up`
  resets unspecified preferences to defaults, which is a common way for this to
  appear without anyone choosing it.
- Do `opus`, `sol`, `nebula` and `legion` currently have `--accept-routes`
  enabled? Verify before the Ansible role is written, so the role codifies the
  intended state rather than the current one.
