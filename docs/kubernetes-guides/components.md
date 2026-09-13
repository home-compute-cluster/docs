# Cluster Architecture: the Components

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

What the control plane and nodes are made of, and how K3s packs all of it into one binary.

Kubernetes splits into a **control plane**, which makes decisions, and **nodes**,
which run workloads. The official component list:

## Control plane components

| Component | What it does |
|---|---|
| **kube-apiserver** | Exposes the Kubernetes HTTP API. Every other component, and `kubectl`, goes through it. |
| **etcd** | Consistent, highly available key value store for all API server data. |
| **kube-scheduler** | Finds Pods not yet bound to a node and assigns each one to a suitable node. |
| **kube-controller-manager** | Runs the controllers that implement API behaviour (Deployments, ReplicaSets, Jobs, node lifecycle, and so on). |
| **cloud-controller-manager** (optional) | Integrates with a cloud provider: node addresses, load balancers, routes. |

## Node components

| Component | What it does |
|---|---|
| **kubelet** | Makes sure the Pods assigned to the node are running, including their containers, and reports status back. |
| **kube-proxy** (optional) | Maintains network rules on the node that implement Services. Some CNI plugins replace it. |
| **Container runtime** | Actually runs containers, through the Container Runtime Interface (CRI). containerd is the common choice. |

## Addons

Addons are ordinary Kubernetes workloads that provide cluster features:

- **DNS** (CoreDNS): cluster-wide name resolution for Services and Pods.
- **Web UI**: dashboards such as the Kubernetes Dashboard or Headlamp.
- **Container resource monitoring**: metrics-server for `kubectl top` and autoscaling,
  Prometheus for everything else.
- **Cluster-level logging**: shipping container logs to a central store.
- **Network plugins**: the CNI that gives Pods their addresses.

## How K3s packages all of this

- **Servers** run `k3s server`: the API server, scheduler, controller manager,
  datastore, plus everything an agent runs.
- **Agents** run `k3s agent`: kubelet, container runtime (containerd) and the CNI
  (flannel), with no datastore or control-plane components.
- K3s includes its own **embedded cloud controller manager**, which handles node
  addresses and runs ServiceLB.
- Agents keep connections open to every server through a **client-side load
  balancer** built into the agent process, so they are not tied to a single server.

**Why this matters in practice.** Because the whole control plane is one process, a
problem with one piece (for example the controller manager losing its leader lease
because etcd is unreachable) exits the whole `k3s` service, kubelet included. That is
exactly what happened on deus in September, and it is covered in the
[control-plane outage postmortem](../postmortems/2026-09-08-deus-control-plane-outage.md).

---

## Sources

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Architecture](https://docs.k3s.io/architecture)
