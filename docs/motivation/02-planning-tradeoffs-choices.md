# The Initial Planning: Tradeoffs and the Choices I Made

*Foundational document. Covers the decisions baked in at the planning stage,
before and during the early build, and the reasoning for each.*

The first document covered *why* a homelab and not the cloud. This one covers the
decisions *inside* that choice: the hardware, the software stack, the
architectural rules, and the sequencing. Almost every one of these was a
tradeoff, and the tradeoff is the interesting part.

## 1. The hardware plan and how it evolved

### The original sketch vs. what I built

The initial concept, captured in the original spec, was a fairly textbook
homelab: one Lenovo Legion laptop as control plane + worker, plus **three budget
N100 mini-PCs** as worker agents, on a S$1,000 budget over 3-6 months. Clean,
symmetric, easy to reason about.

What I actually converged on is messier and better: **three heterogeneous
always-on nodes** using hardware I largely already had or could get cheaply.

- **deus**: Lenovo M910q (i7-6700T class, ~16GB). Always-on control plane. Small,
  low-power, quiet, ideal for the one node that must never go down.
- **nebula**: i7-6700K desktop, 24GB, GTX 1080. Doubles as the ZFS NAS and the
  Docker Compose host (This thing only cost me S$270 and 6 hours of cleaning and
  troubleshooting). 
- **legion**: a laptop with a GTX 1060, battery removed so it runs 24/7 as a
  worker.

Why these names for the machine? Well no real reason, I just find the latin names
more interesting and easier to remember.

### Later additions: sol and opus

Three nodes carried the project through July, but deus was a single point of
failure for the whole control plane. In August I added two refurbished mini PCs:

- **sol**: an Intel NUC with an i5-10210U and 16GB of RAM.
- **opus**: an HP EliteDesk 800 G4 Desktop Mini with an i5-8500.

Both joined as **K3s servers**, not workers. Together with deus that makes three
servers sharing an embedded etcd datastore, which is the smallest setup that can
actually lose a server and keep going. Both were provisioned with Ansible from the
start, so they are the first nodes whose setup is written down as code rather than
remembered.

### The tradeoffs in that shift

- **Symmetric N100 fleet vs. heterogeneous existing hardware.** Symmetric nodes
  are easier to reason about and cheaper to buy new. But heterogeneous hardware I
  already owned was *free*, more powerful, and serves as a better learning experience.
  Real clusters are heterogeneous; learning to place a GPU workload on the one
  node that has a GPU, or to keep replicas off the node that overheats, is a
  *feature* of an uneven cluster. I traded tidiness for realism and zero marginal
  cost.
- **The Singapore pricing reality.** My early assumption that N100 mini-PCs were
  sub-S\$200 didn't survive contact with Singapore retail, where new 16GB/512GB units
  land closer to **S$400**, and the genuinely cheap path is refurbished enterprise 
  USFF boxes (HP EliteDesk, Lenovo ThinkCentre) off Carousell. This is part of why 
  leaning on hardware I already had won: the "cheap mini-PC fleet" wasn't as cheap 
  as the spec implied.
- **N100 mini-PCs stayed on the table** as *additive* nodes if a discount ever put
  them under ~S$200, specifically because they're `linux/amd64` and wouldn't break
  my single-arch image pipeline (unlike ARM Pis). Compatibility drove that
  constraint.

## 2. The software stack: what I chose and what I passed on

The stack was chosen to mirror what these tools look like in a real platform team,
while staying light enough to run on modest hardware.

| Layer | Choice | The main alternative I passed on, and why |
|---|---|---|
| Orchestration | **K3s** | Full kubeadm Kubernetes. Too heavy for this hardware, and K3s is real Kubernetes (same API) with the operational fat trimmed. Single binary, bundled Flannel/Traefik. |
| GitOps | **ArgoCD** | Flux. both are fine; ArgoCD's UI makes the reconciliation loop *visible*, which matters when the goal is learning. |
| Ingress | **Traefik** | nginx-ingress. Traefik ships with K3s and became my single routing brain. NGINX will be used as web servers instead |
| Load balancer | **ServiceLB (bundled with K3s)** | MetalLB. ServiceLB already ships with K3s and exposes a LoadBalancer Service on the host ports of every node, so any node's own IP reaches Traefik. MetalLB would mean running another controller and managing a separate address pool for no real gain on a cluster fronted by a tunnel and a tailnet. |
| Public access | **Cloudflare Tunnel** | Port-forwarding. My ISP router was locked at planning time; the tunnel dials *outbound*, sidestepping inbound ports entirely. |
| Private and remote access | **Tailscale** (WireGuard underneath) | Cloudflare Tunnel or WARP for *everything*. The tunnel hairpins through an edge PoP and is slow for bulk transfer, while a Tailscale subnet router gives SSH, `kubectl` and the lab services from anywhere without opening a port. |
| Storage (block) | **Longhorn** | Rook/Ceph. Far heavier operationally; Longhorn is right-sized for a small cluster with a handful of storage nodes. |
| Storage (bulk/NAS) | **ZFS** | Plain ext4/mdadm. ZFS gives snapshots, checksums, and mirror integrity the NAS role needs. |
| Secrets | **Sealed Secrets** | Vault / External Secrets Operator, overkill until I need dynamic secrets, rotation, or audit logging. |
| Observability | **kube-prometheus-stack** | Rolling my own Prometheus/Grafana. the Helm stack is the industry-standard bundle, they work together and I might as well just use as it is. |
| Database | **CloudNativePG** | Bare Postgres in a Deployment, or Compose Postgres. CNPG gives an operator that owns failover, backups, and WAL archiving. |

### Two stack tradeoffs worth expanding

- **Sealed Secrets over Vault.** Vault is what "real" shops use, so there was a
  pull toward it for the portfolio optics. But it's a heavyweight stateful service
  to run and back up, and I don't need dynamic secrets or rotation yet. Sealed
  Secrets fits GitOps natively, encrypted secrets committed straight to Git, only
  the in-cluster controller can decrypt. The tradeoff I accepted: I *must* back up
  the controller's private key outside the cluster and outside Git, because losing
  it makes every sealed secret unrecoverable. That single backup is the load-
  bearing discipline of the whole choice. I will definitely switch over to Vault
  once a significant number of services warrant its use

- **CNPG over a shared Postgres.** I decided on **one CNPG `Cluster` per
  application** rather than a single shared instance. Shared is cheaper on
  resources; per-app is cleaner blast-radius isolation and matches how you'd
  actually run databases in production. For a learning cluster, the isolation and
  the realism won over the resource saving.

## 3. The architectural rules I locked in

These are the principles that recur through every weekly log. They're the actual
value of the planning, the rules that made later decisions fast.

### Let K3s manage everything it reasonably can

The intent is for K3s to run and reconcile as much of the platform as possible, so
that everything is declared in Git and handled the same way: scheduled, health
checked, restarted and upgraded by the cluster rather than by hand.

What nebula brings is capacity. It is by far the node with the most storage, so it
holds the ZFS `tank` pool and acts as the backup path for most deployments: database
WAL archives and base backups land in Garage on `tank`, and Longhorn backups go to
nebula as well. That makes nebula important, but it is still just a node in the
cluster, not a separate system with its own set of rules.

A few things still run under Docker Compose on nebula, Seafile being the main one.
Seafile went there early because I did not yet trust a week-old cluster with
people's files. That was a reasonable call at the time, not a principle to
preserve, and those services move into K3s when it is worth the effort.

### "Cloudflare Tunnel = dumb pipe, Traefik = routing brain"

All routing logic lives in Traefik IngressRoutes. Cloudflare and Tailscale are
*ingress pipes* that feed Traefik; they carry no routing intelligence. This is
what lets me have multiple ingress paths (public tunnel, private tailnet) without
fragmenting routing across them. The corollary rule: **a single routing brain**
per-app proxies (like nginx fronting an out-of-cluster stack) are allowed *only*
if scoped to that stack, never promoted to a second general-purpose ingress.

### Longhorn cannot use ZFS (We should perhaps help with this implmentation)

A hard technical constraint I learned the hard way: Longhorn needs a **dedicated
disk formatted ext4/XFS at a stable mount path** with `multipathd` blacklisted. It
cannot sit on a ZFS dataset. This shaped the whole storage-hardware plan, I
couldn't just carve Longhorn space out of nebula's ZFS mirror (and ZFS can't
shrink a vdev in place anyway, so clawing space back would mean breaking
redundancy or a full backup-destroy-restore). The clean answer was **buy/acquire
dedicated disks**, which is what I did.

### "2 replicas = meaningful redundancy"

Longhorn below 2 replicas gives no node-independent redundancy. So the target is
2, and since deus overheats (no replicas there), the replica pair is nebula +
legion. The tradeoff accepted: with exactly two targets I have **zero placement
slack**, but two honest replicas beats three where one node is thermally
compromised.

### The GitOps repo conventions

- **Two repos, always**. App source (+ CI) separate from config manifests.
  Single-repo would create an infinite loop, because CI writes the new image tag
  *back* to the config repo, which would re-trigger CI. Splitting them breaks the
  loop. This isn't a style preference; it's structural.
- **Pattern B** (upstream Helm charts): just an `argocd/<svc>.yaml` Application,
  no `apps/<svc>/` folder.
- **`apps/<svc>/` only when I author manifests myself**; multi-source Applications
  when a chart needs my own values + custom manifests.

### The IaC layering boundary

**Ansible owns below-K3s node state; ArgoCD owns in-cluster workloads; the two
must not overlap.** I chose Ansible over Terraform because my manual work is
machine *configuration* (k3s install flags, systemd units, ZFS, sudoers, WoL), not
API-driven *provisioning*, which is Terraform's job and which I mostly don't
have. And I chose to write playbooks **retroactively** (the next time I rebuild a
node) rather than stopping a working build to Ansible-ify it. Bandwidth was the
tradeoff: perfect reproducibility now vs. forward progress now, and progress won.

## 4. Sequencing: the deliberate order of operations

Planning wasn't just *what* to build but *when*, and the ordering was a series of
explicit tradeoffs about managing simultaneous novelty.

- **CI/CD before everything else.** A working deploy loop is the foundation every
  later service rides on, so it came first, even before the storage that some
  services needed.
- **Longhorn deferred until dedicated disks existed.** Rather than fake it on
  `local-path` (which pins pods to nodes) or bodge it onto ZFS (impossible), I
  pinned the few stateful workloads to nodes and waited until I had real disks.
  The tradeoff: temporary node-pinning ugliness vs. doing storage correctly once.
- **Portfolio site before obsync.** Both need Go. But obsync *also* needs Postgres
  *and* a hand-written sync protocol, three new hard things at once. The portfolio 
  site is Go + a few JSON endpoints: one new hard thing. Doing the simpler one first 
  means I learn the deployment shape and the language with low stakes before layering 
  on distributed-systems correctness. This is the clearest example of my core sequencing 
  rule: **don't stack multiple new problems in one step.**
- **Guides written alongside the build**, not after. Turning each hard-won setup
  into a reusable guide (CI/CD, Docker, K3s, observability) meant the *next*
  service was a checklist, not a re-derivation.

## 5. What I consciously left open

Good planning also means naming what you *don't* decide. Carried forward
deliberately:

- **Control-plane HA**: deus was a known single point of failure. I decided *not*
  to chase HA prematurely: two etcd servers is worse than one (no quorum), real HA
  needs three. I held off until I could buy two nodes at once, and in August sol
  and opus joined as the second and third servers. This one is now closed.
- **KEDA / autoscaling**: gated behind Traefik metrics existing first.
- **Renovate**: planned for dependency PRs, not yet stood up.

Leaving these explicitly open, with the reasoning attached, is itself a planning
choice. It keeps the scope honest and stops me from forcing premature answers to
questions the cluster isn't ready to settle.
