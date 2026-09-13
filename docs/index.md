# Home Compute Cluster

*A five-node bare-metal K3s platform, built to learn how real infrastructure is actually operated.*

Not a toy repo, not a cloud tutorial. A standing homelab where GitOps, distributed
storage, observability, and hand-rolled networking run on hardware I own, so the
mistakes are cheap and the lessons are real.

[Read the motivation](motivation/01-motivation.md){ .md-button .md-button--primary }
[Jump to architecture](system-architecture/home-cluster.md){ .md-button }

---

## The rundown

<div class="grid cards" markdown>

-   :material-target:{ .lg .middle } __Why this exists__

    ---

    The career and learning case for owning the hardware, and the alternatives
    (cloud, single box, Pi cluster) that lost the argument.

    [:octicons-arrow-right-24: Motivation](motivation/01-motivation.md)

-   :material-sitemap:{ .lg .middle } __Architecture__

    ---

    Node inventory, network topology, GitOps control loop, and the split
    public / tailnet ingress model, with diagrams.

    [:octicons-arrow-right-24: System design](system-architecture/home-cluster.md)

-   :material-source-branch:{ .lg .middle } __CI / CD__

    ---

    The `git push -> build -> deploy` loop. GitHub Actions to ghcr.io, ArgoCD
    reconciling the cluster to match Git.

    [:octicons-arrow-right-24: Pipeline](CICD/cicd-pipeline-setup-guide.md)

-   :material-kubernetes:{ .lg .middle } __K3s & containers__

    ---

    How the cluster is stood up and how workloads get packaged and scheduled
    across heterogeneous nodes.

    [:octicons-arrow-right-24: K3s guide](kubernetes-guides/k3s-kubernetes-guide.md)

-   :material-lightbulb-on-outline:{ .lg .middle } __Lessons learnt__

    ---

    What building and breaking the cluster actually taught, grouped by topic:
    GitOps, secrets, networking, storage, the control plane and debugging.

    [:octicons-arrow-right-24: Lessons](lessons/index.md)

-   :material-fire-alert:{ .lg .middle } __Postmortems__

    ---

    Full write-ups of the real failures: timelines, evidence, root causes and
    the fixes that came out of them.

    [:octicons-arrow-right-24: Postmortems](postmortems/index.md)

</div>

---

## The rig

Five heterogeneous, always-on x86 nodes. Three of them run the K3s control plane
with embedded etcd, so losing any single server no longer takes the cluster down.
The names stuck, so they stayed.

| Node | Callsign | Hardware | Role |
|------|----------|----------|------|
| **deus** | control plane | Lenovo M910q, i7-6700T | K3s server, etcd member, Tailscale subnet router |
| **sol** | control plane | Intel NUC, i5-10210U, 16GB | K3s server, etcd member |
| **opus** | control plane | HP EliteDesk 800 G4 Mini, i5-8500 | K3s server, etcd member |
| **nebula** | worker + storage | i7-6700K, GTX 1080 | Most storage in the cluster, ZFS, backup target, Longhorn replica |
| **legion** | worker | Laptop, GTX 1060 | Longhorn replica |

**Stack:** K3s, ArgoCD, Traefik, ServiceLB, kube-vip, Longhorn, ZFS, Prometheus,
Grafana, Cloudflare Tunnel, Tailscale

**Domains:** `*.packetcraft.dev` (public) and `*.lab.packetcraft.dev` (tailnet-only)

---

## Where to start

New here? Read the [motivation](motivation/01-motivation.md) for the why, then the
[architecture](system-architecture/home-cluster.md) for the how. If you just want
to follow the build as it happened, the [weekly logs](weekly/2026-05-10.md) are the
running diary.

> The value is in the decision log, not the uptime number. Every choice here has a
> reason, and the reason is written down.
