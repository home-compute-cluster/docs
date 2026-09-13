# Lessons: Control Plane and Nodes

The cluster went from one K3s server to three in late August, and the first two
weeks after that produced three postmortems. These are the lessons about running
the control plane and the machines under it.

---

## 1. A single K3s server quietly runs on SQLite

**What happened.** I had assumed the cluster used etcd from the start. With one
server and two agents, K3s had been using its embedded SQLite datastore all along,
and I only found out when adding sol and opus in Week 17. SQLite cannot back more
than one server, so the datastore had to be migrated first.

**The lesson.** Know what your control plane is actually storing its state in. You
can check on a server:

```bash
ls /var/lib/rancher/k3s/server/db/
# state.db*  means SQLite
# etcd/      means embedded etcd
```

**What I do now.** The migration is a restart of the existing server with
`--cluster-init`, after which new servers join with `--server https://<first>:6443`.
All servers must share the same networking and feature flags, so they now come from
one Ansible inventory.

## 2. Two servers is worse than one

**What happened.** In Week 10 I was about to buy a single mini PC as a second server.
An etcd cluster of two needs both members to agree, so either one failing stops
writes. That doubles the chance of an outage instead of halving it.

**The lesson.** etcd needs an odd number of members. Three is the smallest cluster
that survives losing one.

## 3. Three servers survive one failure, not one failure plus maintenance

**What happened.** While opus was drained and stopped to debug its reset loop, etcd
ran on two of three members for about half a day. During the September outage it ran
on two of three for nearly four days. In both cases one more failure would have meant
restoring from a snapshot.

**The lesson.** Taking a server down on purpose uses up the fault tolerance you
bought. It should be a planned, short window, not a background state.

**What I do now.** Before taking a server offline, check that the other two are
healthy. If a server is going to be out for long, treat the cluster as fragile until
it is back.

## 4. In K3s, one component losing its lease takes down the whole node

**What happened.** When deus lost contact with its etcd peers, the controller
manager lost its leader lease and `k3s.service` exited. Because K3s runs the API
server, scheduler, controller manager, kubelet and etcd in one process, all of them
went down together.

**The lesson.** This is a deliberate K3s trade for simplicity, but it means network
faults on a server node have much less room for error than on a cluster with
separate components.

## 5. Monitoring defaults assume upstream Kubernetes

**What happened.** kube-prometheus-stack's etcd job looks for pods labelled
`component=etcd` in `kube-system`. Under K3s, etcd is inside the `k3s` binary, so the
job had zero targets and nobody noticed. K3s also does not expose etcd metrics unless
`etcd-expose-metrics: true` is set. The result was nearly four days of degraded quorum
without a single alert.

**The lesson.** A rule that syncs green is not a rule that works. Change the
distribution and the defaults may be watching things that do not exist.

**What I am changing.** Enable etcd metrics, scrape them explicitly, and alert on
`etcd_server_has_leader == 0` and on member count. This is a P0 item from the
September postmortem and is not done yet. Once it is, the alert gets tested by
actually stopping a member, because that is the only proof it fires.

## 6. Alert on the things you keep finding by eye

**What happened.** Every incident so far was found by a person: a gap in a Grafana
panel, a `kubectl` call that timed out. The opus reset loop ran for about two days
before I spotted a pattern of gaps.

**What I am adding.** Alerts for what I found manually (still open at the time of
writing):

- Unexpected reboots: `changes(node_boot_time_seconds[1h]) > 0`
- `k3s.service` restarts: needs node-exporter's `--collector.systemd`
- A node disappearing from scrapes: `up == 0` for a few minutes

## 7. Anything configured by hand has an unknown value

**What happened.** The Tailscale flags and the `ufw` state on deus were both set by
hand, drifted from sol and opus, and caused the September outage. Nothing could have
detected the drift because there was no recorded intended state to compare with.

**The lesson.** Automating the last two nodes with Ansible made them reproducible.
The parts left manual (Tailscale was "too complicated to automate") are exactly the
parts that broke.

**What I am changing.** Tailscale settings and firewall state are being moved into
the Ansible roles as asserted state, applied on every run. This is the top action
item from the September outage. When writing a role, capture the intended state, not
whatever the nodes currently happen to have.

## 8. Put cheap safety nets on every node

**What happened.** The deus lockup on 2 September needed someone to physically press
the power button, 21 minutes after it died.

**The baseline I want on every node**, going into the Ansible base role:

- `RuntimeWatchdogSec=60` in `/etc/systemd/system.conf` with a hardware watchdog
  driver loaded, so a hard hang becomes a reboot within a minute.
- `sysstat` installed, because its on-disk samples survive hangs that journald
  loses.
- Persistent journald, so an investigation can start after the reboot.
- `kured` to handle the cordon, drain, reboot and uncordon cycle when OS updates need
  a restart, one node at a time. This one was deployed in September and is still
  being watched to confirm it behaves.

## 9. One node doing five jobs is five single points of failure

**What happened.** deus is a control-plane node, an etcd member, a schedulable worker,
the kube-vip VIP holder and the only Tailscale subnet router. Both deus incidents
turned a single-node problem into a whole-cluster access problem because of that last
role.

**What I am changing.** Spreading the roles out: a second subnet router, and a
taint on deus to keep general workloads off the node that carries the most critical
duties. Both are open action items.

## 10. Match the hardware to the job, not the spec sheet

**What happened.** In Week 1 the laptop looked like the obvious control plane because
it was the most powerful machine. The M910q was the better choice: built for 24/7,
low idle power, no lid or battery to worry about. Later, deus's NVMe running near 90
degrees at idle is why it holds no Longhorn replicas.

**The lesson.** Control-plane nodes want reliability and low idle power. Storage
nodes want cool, NAS-rated disks. Raw horsepower rarely decides anything.

**Related, from the postmortems.** None of the mini PCs have ECC memory, so memory
faults on them cannot be detected at all. That is worth weighing when buying the next
node.
