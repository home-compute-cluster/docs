# K3s / Kubernetes: Working Reference

A practical guide to Kubernetes, oriented around K3s. It is written from the seat of
someone running a small, mixed cluster, so where the real cluster made a concept
concrete, that context stays in. Everything here is standard Kubernetes unless a
section is explicitly marked as K3s-specific.

Facts about how Kubernetes and K3s behave are taken from the official documentation
at kubernetes.io and docs.k3s.io. Each page ends with the sources it relies on, and
the full list is at the bottom of this page, so you can check anything that matters
before relying on it.

The cluster this guide refers to has five nodes:

- **Three servers** (deus, sol, opus): run the K3s control plane and share an
  embedded etcd datastore. They also run ordinary workloads.
- **Two agents** (nebula, legion): workers only. nebula has most of the storage;
  legion is a laptop with a small GPU.

Substitute your own node names freely.

---

## How this guide is organised

Start here for the big picture, then jump to whichever area you need.

| Page | What it covers |
|---|---|
| [Components](components.md) | Control plane and node components, addons, and how K3s packages them |
| [Objects and workloads](objects-and-workloads.md) | Object hierarchy, manifest anatomy, Deployments, StatefulSets, Jobs, sidecars, HPA, PDBs, QoS, probes |
| [kubectl](kubectl.md) | Everyday commands, the SSH port-forward trap, and a quick reference card |
| [Scheduling](scheduling.md) | nodeSelector, affinity, topology spread, taints and tolerations, requests and limits |
| [Networking](networking.md) | Network model, flannel, Services and kube-proxy, DNS, Ingress, Gateway API, ServiceLB, NetworkPolicy, ports |
| [Storage](storage.md) | PVCs, StorageClasses, access modes, local-path versus Longhorn |
| [Configuration and security](configuration-and-security.md) | ConfigMaps, Secrets, RBAC, Pod Security Admission, securityContext |
| [Packaging and patterns](packaging-and-patterns.md) | Helm, Kustomize, K3s HelmChartConfig, common workload shapes, an end-to-end example |
| [Deployment modes](deployment-modes.md) | Ways to run Kubernetes, K3s cluster shapes and datastores, choosing between them |
| [K3s operations](k3s-operations.md) | Config file, packaged components, etcd snapshots and restore, upgrades, node lifecycle, observability |
| [Debugging](debugging.md) | A playbook for Pods, Services, networking and the control plane |

---

## K3s vs Kubernetes: settle it first

K3s **is** Kubernetes. It is a CNCF-certified conformant distribution, so every API
object, every `kubectl` command and every manifest works the same way on upstream
Kubernetes, EKS, GKE or a 500-node cluster. What K3s changes is *packaging and
operations*, not the API:

| Concern | Typical upstream setup | K3s |
|---|---|---|
| Install | kubeadm plus separately installed pieces | one binary, one `curl \| sh` |
| Control plane processes | separate API server, scheduler, controller manager (often static pods) | all inside the single `k3s server` process |
| Datastore | etcd | SQLite for a single server, embedded etcd for HA, or an external SQL database or etcd |
| CNI (pod networking) | you choose and install one | flannel, bundled |
| Ingress controller | you install one | Traefik, bundled |
| Storage class | you install one | `local-path`, bundled |
| LoadBalancer Services | cloud provider | ServiceLB (Klipper LB), bundled |
| NetworkPolicy | depends on the CNI | embedded network policy controller |
| Metrics for `kubectl top` | you install metrics-server | metrics-server, bundled |

**Takeaway:** describe it as "K3s, a conformant Kubernetes distribution". The skills
transfer directly.

---

## The mental model

Kubernetes is a **reconciliation engine**. You declare desired state (YAML
manifests), and controllers keep working to make actual state match. You almost never
tell it to *do* something. You tell it what should *be true*, and it works out the
actions. GitOps (for example ArgoCD with `selfHeal`) is the same idea one layer up:
the GitOps controller reconciles Git into the cluster, while Kubernetes reconciles
manifests into running containers.

What happens when you `kubectl apply` a Deployment:

```
you --apply--> API server --stores--> datastore (etcd or SQLite)
                   |
                   | controllers watch the API server
                   v
      Deployment controller:  "I need 1 ReplicaSet"
      ReplicaSet controller:  "I need 1 Pod"
      Scheduler:              "that Pod goes on node X"
                   |
                   v
      kubelet on node X:      "pull the image, start the container"
```

Nothing pushes work to nodes. Each node's **kubelet** watches the API server for Pods
assigned to it and makes them run. Every component talks to the API server, and only
the API server talks to the datastore.

---

## References

Kubernetes documentation (kubernetes.io):

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)

K3s documentation (docs.k3s.io):

- [Architecture](https://docs.k3s.io/architecture)
- [Requirements (inbound ports)](https://docs.k3s.io/installation/requirements)
- [Cluster Datastore](https://docs.k3s.io/datastore)
- [High Availability Embedded etcd](https://docs.k3s.io/datastore/ha-embedded)
- [Basic Network Options (flannel)](https://docs.k3s.io/networking/basic-network-options)
- [Networking Services (CoreDNS, Traefik, ServiceLB, network policy)](https://docs.k3s.io/networking/networking-services)
- [Managing Packaged Components](https://docs.k3s.io/installation/packaged-components)
- [k3s server CLI reference](https://docs.k3s.io/cli/server)
- [etcd-snapshot](https://docs.k3s.io/cli/etcd-snapshot)
- [Automated Upgrades](https://docs.k3s.io/upgrades/automated)

Other:

- [kube-vip on K3s](https://kube-vip.io/docs/usage/k3s/)
