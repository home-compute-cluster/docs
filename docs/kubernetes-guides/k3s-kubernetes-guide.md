# K3s / Kubernetes: Working Reference

A practical guide to Kubernetes, oriented around K3s. It is written from the seat of
someone running a small, mixed cluster, so where the real cluster made a concept
concrete, that context stays in. Everything here is standard Kubernetes unless a
section is explicitly marked as K3s-specific.

Facts about how Kubernetes and K3s behave are taken from the official documentation
at kubernetes.io and docs.k3s.io. The [references](#references) at the end list the
exact pages, so you can check anything that matters before relying on it.

The cluster this guide refers to has five nodes:

- **Three servers** (deus, sol, opus): run the K3s control plane and share an
  embedded etcd datastore. They also run ordinary workloads.
- **Two agents** (nebula, legion): workers only. nebula has most of the storage;
  legion is a laptop with a small GPU.

Substitute your own node names freely.

---

## 0. K3s vs Kubernetes: settle it first

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

## 1. The mental model

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

## 2. Cluster architecture: the components

Kubernetes splits into a **control plane**, which makes decisions, and **nodes**,
which run workloads. The official component list:

### Control plane components

| Component | What it does |
|---|---|
| **kube-apiserver** | Exposes the Kubernetes HTTP API. Every other component, and `kubectl`, goes through it. |
| **etcd** | Consistent, highly available key value store for all API server data. |
| **kube-scheduler** | Finds Pods not yet bound to a node and assigns each one to a suitable node. |
| **kube-controller-manager** | Runs the controllers that implement API behaviour (Deployments, ReplicaSets, Jobs, node lifecycle, and so on). |
| **cloud-controller-manager** (optional) | Integrates with a cloud provider: node addresses, load balancers, routes. |

### Node components

| Component | What it does |
|---|---|
| **kubelet** | Makes sure the Pods assigned to the node are running, including their containers, and reports status back. |
| **kube-proxy** (optional) | Maintains network rules on the node that implement Services. Some CNI plugins replace it. |
| **Container runtime** | Actually runs containers, through the Container Runtime Interface (CRI). containerd is the common choice. |

### Addons

Addons are ordinary Kubernetes workloads that provide cluster features:

- **DNS** (CoreDNS): cluster-wide name resolution for Services and Pods.
- **Web UI**: dashboards such as the Kubernetes Dashboard or Headlamp.
- **Container resource monitoring**: metrics-server for `kubectl top` and autoscaling,
  Prometheus for everything else.
- **Cluster-level logging**: shipping container logs to a central store.
- **Network plugins**: the CNI that gives Pods their addresses.

### How K3s packages all of this

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

## 3. The object hierarchy: what actually runs your stuff

From smallest to largest, the chain that turns an image into a running thing:

- **Container**: your image (`registry/app:tag`), running.
- **Pod**: one or more containers sharing a network namespace and volumes. The
  *smallest schedulable unit*. Pods are disposable: they get replaced, not repaired,
  and get a new IP each time.
- **ReplicaSet**: keeps N identical Pods alive. You rarely touch these directly.
- **Deployment**: manages ReplicaSets to give rolling updates and rollbacks. **This is
  what you write** for stateless services.
- **StatefulSet**: like a Deployment, but Pods get stable names (`db-0`, `db-1`) and
  their own persistent volumes. For databases and anything where identity matters.
- **DaemonSet**: one Pod on every node, or every matching node. For node-level agents:
  node-exporter, log shippers, CNI pods, device plugins, ServiceLB's `svclb` pods.
- **Job / CronJob**: run-to-completion work and scheduled work. Backups, batch jobs,
  migrations.

Supporting objects:

- **Service**: a stable virtual IP and DNS name in front of a changing set of Pods.
  Covered in depth in section 8.
- **Namespace**: a logical partition (`argocd`, `monitoring`, one per app). Scopes
  names, RBAC, quotas and network policies.
- **ConfigMap / Secret**: configuration and credentials injected into Pods.
- **PersistentVolume (PV) / PersistentVolumeClaim (PVC)**: the storage abstraction.
- **Ingress / Gateway / IngressRoute**: HTTP routing from outside into Services.
- **CustomResourceDefinition (CRD)**: lets controllers add their own kinds, such as
  ArgoCD's `Application`, CNPG's `Cluster` or Traefik's `IngressRoute`.

### Metadata that holds it together

- **Labels** are key value pairs used for selection. Services, Deployments, network
  policies and node selectors all find things by label.
- **Annotations** are key value pairs for tools and humans, not for selection.
- **Owner references** record which object created which. A ReplicaSet is owned by
  its Deployment, a Pod by its ReplicaSet. Deleting an owner lets the garbage
  collector clean up what it owned.
- **Finalizers** block deletion until a controller has done its cleanup. An object
  stuck in `Terminating` usually has a finalizer whose controller is gone.

---

## 4. Workload resources in more depth

### Deployments and rollout strategy

A Deployment supports two update strategies:

- **RollingUpdate** (the default): replaces Pods gradually. `maxSurge` (default 25%)
  is how many extra Pods may exist during the update, and `maxUnavailable` (default
  25%) is how many may be missing.
- **Recreate**: kills every old Pod before starting new ones. Causes downtime, but
  it is the right choice when two versions must never run at once, for example an app
  holding a `ReadWriteOnce` volume.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0     # never drop below the desired count
```

### StatefulSets

Use a StatefulSet when each replica needs its own identity or its own disk. Pods are
created in order (`app-0`, then `app-1`), keep their names across restarts, and each
gets its own PVC from `volumeClaimTemplates`. A StatefulSet is normally paired with a
**headless Service** (`clusterIP: None`) so each Pod gets a stable DNS name.

In practice, for databases, an operator such as CloudNativePG is usually better than
writing a StatefulSet yourself, because the operator also handles failover, backups
and WAL archiving.

### Jobs and CronJobs

A Job runs Pods until a set number complete successfully. A CronJob creates Jobs on a
schedule. Two settings worth knowing: `backoffLimit` (retries before the Job is marked
failed) and, on CronJobs, `concurrencyPolicy` (`Allow`, `Forbid` or `Replace`) so a
slow run does not overlap with the next one.

### Init containers and sidecars

- **Init containers** run to completion, in order, before the app containers start.
  Good for waiting on a dependency, running a migration or cloning a repo into a
  shared volume.
- **Sidecar containers** run alongside the app for the whole life of the Pod (log
  shippers, proxies). Since Kubernetes 1.29 they are defined natively as init
  containers with `restartPolicy: Always`, which means they start before the app and
  keep running. The feature has been stable since 1.33.

```yaml
spec:
  initContainers:
    - name: log-shipper
      image: alpine:3.20
      restartPolicy: Always          # this makes it a sidecar
      command: ["sh", "-c", "tail -F /var/log/app/app.log"]
  containers:
    - name: app
      image: registry/app:1.0
```

### Horizontal Pod Autoscaler (HPA)

An HPA changes the replica count of a Deployment or StatefulSet based on metrics. The
core formula is:

```
desiredReplicas = ceil( currentReplicas * currentMetricValue / desiredMetricValue )
```

Two requirements trip people up. First, it needs a metrics API: for CPU and memory
that is metrics-server, which K3s bundles. Second, for utilisation targets the
containers **must have resource requests set**, because utilisation is measured as a
percentage of the request. HPAs do not apply to DaemonSets.

### PodDisruptionBudgets (PDBs)

Kubernetes separates two kinds of disruption:

- **Voluntary**: a person or tool chose it. Draining a node, deleting a Pod,
  updating a Deployment.
- **Involuntary**: hardware failure, a kernel panic, a node falling off the network,
  eviction because a node ran out of resources.

A PDB limits how many Pods of an application can be down at once from **voluntary**
disruptions, using `minAvailable` or `maxUnavailable`. It is only enforced through
the **Eviction API**, which is what `kubectl drain` and tools like kured use. Deleting
a Pod or Deployment directly bypasses it, and involuntary disruptions cannot be
prevented by it (though they still count against the budget).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: traefik
  namespace: kube-system
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: traefik
```

A PDB only helps if there is more than one replica. With `replicas: 1` and
`maxUnavailable: 0`, a drain will simply wait forever.

### Quality of Service (QoS) classes

Every Pod gets a QoS class from its requests and limits:

| Class | Rule | Evicted |
|---|---|---|
| **Guaranteed** | every container has CPU and memory requests and limits, and each limit equals its request | last |
| **Burstable** | not Guaranteed, but at least one container has a CPU or memory request or limit | second |
| **BestEffort** | no requests or limits anywhere | first |

When a node runs short of memory or disk, the kubelet evicts BestEffort Pods first,
then Burstable, then Guaranteed. Anything you care about should at least have
requests.

---

## 5. kubectl: the commands you will actually run

The pattern is almost always `kubectl <verb> <resource> [name] [flags]`.

### Inspection

```bash
kubectl get nodes -o wide              # all nodes, IPs, K3s version, OS
kubectl get pods -A                    # -A = all namespaces
kubectl get pods -n <ns> -o wide       # which node each pod landed on
kubectl get all -n <ns>                # most common objects in one namespace
kubectl get deployment,svc,pvc -n <ns>

kubectl describe pod <pod> -n <ns>     # events: why it will not schedule, image pull errors
kubectl describe node <node>           # capacity, taints, allocated resources

kubectl logs <pod> -n <ns>             # stdout and stderr
kubectl logs <pod> -n <ns> -f          # follow
kubectl logs <pod> -n <ns> --previous  # logs from the container that crashed
```

### Interacting with running Pods

```bash
kubectl exec -it <pod> -n <ns> -- sh
kubectl exec <pod> -n <ns> -- nginx -t
kubectl cp <pod>:/etc/nginx/nginx.conf ./out.conf -n <ns>
kubectl debug -it <pod> -n <ns> --image=busybox:1.36   # temporary debug container
```

### Port-forwarding, and the SSH trap

```bash
kubectl port-forward -n <ns> svc/<svc> 8888:80
```

If you run this **inside an SSH session** on a node, `localhost:8888` is bound on
*that node*, not on your laptop. Either tunnel it back (`ssh -L 8888:localhost:8888
<node>`), or bind to all interfaces (`--address 0.0.0.0`) and use the node's IP.

The clean long-term fix is to run `kubectl` on your own machine. Copy
`/etc/rancher/k3s/k3s.yaml` from a server, change `server:` to an address the API
certificate covers (on this cluster, the kube-vip address
`https://192.168.1.241:6443`), and `localhost` means your laptop again.

### Applying and deleting

```bash
kubectl apply -f deployment.yaml       # create or update to match the file
kubectl apply -k apps/<svc>/           # apply a Kustomize directory
kubectl apply --server-side -f big-crd.yaml
kubectl delete -f deployment.yaml
kubectl delete pod <pod> -n <ns>       # its controller recreates it: a quick restart
```

**Prefer `apply` over `create` and `edit`.** Everything should be reproducible from
files. With GitOps self-heal, imperative changes to managed resources get reverted
anyway.

### Rollouts and scaling

```bash
kubectl rollout status deployment/<svc> -n <ns>
kubectl rollout history deployment/<svc> -n <ns>
kubectl rollout undo deployment/<svc> -n <ns>
kubectl rollout restart deployment/<svc> -n <ns>
kubectl scale deployment/<svc> -n <ns> --replicas=3
```

### Quality of life

```bash
kubectl get pods -A -w                                  # watch live
kubectl get events -n <ns> --sort-by=.lastTimestamp     # what just happened
kubectl api-resources                                   # every kind, short name, API group
kubectl explain deployment.spec.strategy                # schema docs from the live cluster
kubectl auth can-i create deployments -n <ns> --as=system:serviceaccount:<ns>:<sa>
alias k=kubectl
```

When two operators define the same short name (Longhorn and CNPG both have a
`backup`), use the full resource name: `kubectl get backups.postgresql.cnpg.io`.

---

## 6. Anatomy of a manifest

Every object has the same four top-level keys:

```yaml
apiVersion: apps/v1     # API group and version that owns this kind
kind: Deployment        # the object type
metadata:
  name: web             # unique within namespace + kind
  namespace: web
  labels: { app: web }
spec:                   # DESIRED state: the part you design
  replicas: 2
  selector:             # how this Deployment finds its Pods
    matchLabels: { app: web }
  template:             # the Pod blueprint
    metadata:
      labels: { app: web }   # the selector matches THIS
    spec:
      containers:
        - name: web
          image: registry/web-app:1.4.2
          ports:
            - containerPort: 80
# status:               # ACTUAL state: written by Kubernetes, never by you
```

The **label and selector** coupling is the thing that trips everyone up. A Service
finds Pods by selector; a Deployment manages Pods by selector. If they do not match,
you get a Service with no endpoints or a Deployment that manages nothing.

---

## 7. Scheduling: placement in a mixed cluster

The scheduler decides placement **once, when the Pod is created**, based on declared
`resources.requests` (not actual usage) and the constraints below. There is no
automatic rebalancing afterwards.

### nodeSelector: simplest, exact match

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: nebula
```

### Node labels: your own taxonomy

```bash
kubectl label node nebula role=storage
kubectl label node legion gpu=true
kubectl get nodes --show-labels
```

`nodeSelector: { role: storage }` says what you mean and survives replacing the
hardware.

### Affinity and anti-affinity

`nodeSelector` is exact and AND-only. Affinity adds OR, soft preferences and rules
relative to other Pods:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: gpu
                operator: In
                values: ["true"]
    podAntiAffinity:                       # spread replicas across nodes
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels: { app: web }
            topologyKey: kubernetes.io/hostname
```

### Topology spread constraints

For "keep replicas evenly spread", topology spread constraints are more direct than
anti-affinity:

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels: { app: web }
```

### Taints and tolerations: the node says no by default

A **taint** repels Pods unless they explicitly **tolerate** it. Typical uses: keeping
general workloads off control-plane nodes, or reserving a special node.

```bash
kubectl taint node deus node-role.kubernetes.io/control-plane=true:NoSchedule
```

```yaml
spec:
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
```

Effects: `NoSchedule` (no new Pods), `PreferNoSchedule` (soft), `NoExecute` (also
evicts running Pods that do not tolerate it).

**A toleration is permission, not attraction.** A Pod that tolerates a taint can land
on that node, but nothing makes it go there. To route a GPU workload to a tainted GPU
node you need both a toleration (permission) and a nodeSelector or affinity
(attraction).

### Resource requests and limits

```yaml
containers:
  - name: web
    resources:
      requests:           # reserved; used for placement and QoS
        cpu: 100m         # 0.1 core
        memory: 128Mi
      limits:             # ceiling; CPU is throttled, memory over the limit is OOM-killed
        cpu: 500m
        memory: 256Mi
```

Without requests, the scheduler assumes the Pod needs almost nothing and can overpack
a 16GB node.

---

## 8. Networking in Kubernetes

This is the part that confuses most people, because several layers each solve a
different problem. Working from the inside out:

```
Pod IPs (CNI)  ->  Services (virtual IPs, kube-proxy)  ->  DNS (CoreDNS)
               ->  getting traffic in (NodePort, LoadBalancer, Ingress, Gateway)
               ->  restricting traffic (NetworkPolicy)
```

### 8.1 The Kubernetes network model

The rules every conformant cluster follows:

- **Every Pod gets its own cluster-wide IP address.** Containers inside one Pod share
  that address and can reach each other on `localhost`.
- **All Pods can reach all other Pods**, on the same node or different nodes,
  **without NAT or proxies.**
- **Agents on a node** (the kubelet, system daemons) can reach every Pod on that node.

Kubernetes defines these rules but does not implement them itself. That is the job of
the network plugin.

### 8.2 Pod networking: CNI and flannel

A **CNI plugin** gives each Pod its address and makes Pod-to-Pod traffic work across
nodes. Common plugins are flannel, Calico and Cilium.

K3s bundles **flannel**, which gives each node a slice of the Pod address range and
connects the nodes together. flannel has several backends:

| Backend | How it works | When to use |
|---|---|---|
| `vxlan` (default) | Encapsulates Pod traffic in VXLAN over UDP 8472 | Works on almost any network |
| `host-gw` | Plain IP routes to each node's Pod subnet via that node's IP | Faster, but needs all nodes on the same layer 2 network |
| `wireguard-native` | Encapsulates and encrypts traffic with WireGuard (UDP 51820, and 51821 for IPv6) | Nodes on untrusted networks |
| `none` | Disables flannel | When installing your own CNI |

K3s defaults worth knowing (from the `k3s server` reference):

| Setting | Default |
|---|---|
| Pod network (`--cluster-cidr`) | `10.42.0.0/16` |
| Service network (`--service-cidr`) | `10.43.0.0/16` |
| Cluster DNS address (`--cluster-dns`) | `10.43.0.10` |
| Cluster domain (`--cluster-domain`) | `cluster.local` |
| NodePort range | `30000-32767` |

These ranges are separate from your LAN (`192.168.1.0/24` here), and they must not
overlap with it or with anything you route to, such as a VPN.

To use a different CNI, start K3s with `--flannel-backend=none` and usually
`--disable-network-policy`, then install the CNI yourself. Dual-stack (IPv4 and IPv6)
has to be configured when the cluster is first created; it cannot be added later.

### 8.3 Services: stable addresses for moving Pods

Pods come and go, so clients should never talk to Pod IPs directly. A **Service**
gives a stable virtual IP and DNS name, and forwards to whichever Pods currently match
its selector.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: web
spec:
  selector:
    app: web
  ports:
    - port: 80          # the Service's port
      targetPort: 8080  # the container's port (defaults to the same as port)
```

**Service types:**

| Type | What you get |
|---|---|
| **ClusterIP** (default) | A virtual IP reachable only inside the cluster. Most Services should be this. |
| **NodePort** | ClusterIP, plus a port (30000-32767 by default) opened on every node's IP. |
| **LoadBalancer** | NodePort and ClusterIP, plus an external address from a load balancer implementation. |
| **ExternalName** | No proxying at all. DNS returns a CNAME to an external name. |

**Headless Services** (`clusterIP: None`) have no virtual IP. DNS returns the Pod IPs
directly, which is what StatefulSets use to give each Pod a stable name.

**EndpointSlices** are how a Service knows its backends. A controller watches for Pods
that match the selector and keeps the Service's EndpointSlices up to date. If a
Service has no endpoints, the selector matches nothing or the Pods are not ready:

```bash
kubectl get endpointslices -n <ns> -l kubernetes.io/service-name=<svc>
```

**kube-proxy** makes the virtual IPs work. It watches Services and EndpointSlices and
programs each node to catch traffic sent to a Service IP and redirect it to one of the
endpoints. Its modes on Linux:

- **`iptables`**: the current default. Picks a backend at random for each connection.
- **`ipvs`**: uses the kernel's IPVS load balancer.
- **`nftables`**: uses nftables. The Kubernetes docs say a future version will make
  this the default, and recommend setting the mode explicitly so an upgrade does not
  change it unexpectedly.

In K3s, kube-proxy runs inside the `k3s` process rather than as a DaemonSet.

**Traffic policies** control where traffic can go:

- `externalTrafficPolicy: Local` on a NodePort or LoadBalancer Service only sends
  traffic to endpoints on the node that received it. This keeps the client's source
  IP, but a node with no local endpoint will not answer.
- `internalTrafficPolicy: Local` does the same for traffic from inside the cluster.

### 8.4 DNS: finding Services by name

CoreDNS (in `kube-system`) gives Services and Pods DNS names:

| Record | Name | Resolves to |
|---|---|---|
| Normal Service | `<svc>.<ns>.svc.cluster.local` | the ClusterIP |
| Headless Service | `<svc>.<ns>.svc.cluster.local` | the IPs of all selected Pods |
| Named port | `_<port>._<proto>.<svc>.<ns>.svc.cluster.local` | SRV record with port and target |
| StatefulSet Pod (with headless Service) | `<pod>.<svc>.<ns>.svc.cluster.local` | that Pod's IP |

The kubelet writes each Pod's `/etc/resolv.conf` with search domains, so short names
work:

```
nameserver 10.43.0.10
search <ns>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

From a Pod in the same namespace, `web` works. From another namespace, use
`web.<ns>`. The `ndots:5` option means any name with fewer than five dots is tried
against the search domains first, which is why lookups of external names can be slow
or odd inside Pods.

A Pod's `dnsPolicy` changes this:

- `ClusterFirst` (the default): cluster DNS first, then the node's upstream servers.
- `Default`: use the node's DNS settings instead of cluster DNS.
- `ClusterFirstWithHostNet`: what a `hostNetwork: true` Pod needs to still use
  cluster DNS.
- `None`: ignore everything and use `dnsConfig` from the Pod spec.

### 8.5 Getting traffic into the cluster

There are several layers of increasing capability.

**NodePort.** Every node answers on a high port. Fine for testing, awkward for
anything real.

**LoadBalancer.** In a cloud, this creates a cloud load balancer. On bare metal,
something in the cluster has to provide the address. The common options:

- **ServiceLB** (bundled with K3s): uses each node's own IP and host ports. No extra
  addresses needed. Covered in 8.6.
- **MetalLB**: hands out addresses from a pool you reserve on the LAN, announced with
  ARP or BGP.
- **kube-vip**: can provide a floating virtual IP for the control plane, and in
  services mode can also give LoadBalancer Services their addresses.

**Ingress.** An Ingress routes HTTP and HTTPS by host name and path to Services, and
can terminate TLS. It does nothing on its own: **an Ingress controller must be
running** to act on it. Path matching uses `pathType` of `Prefix`, `Exact` or
`ImplementationSpecific`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: web
spec:
  ingressClassName: traefik
  rules:
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

The Ingress API is **frozen**: it stays generally available and will not be removed,
but it gets no new features. The Kubernetes project recommends **Gateway API** for new
work.

**Gateway API.** An add-on installed as CRDs, supported by many controllers (Traefik
included). It splits responsibility by role:

| Kind | Owned by | What it defines |
|---|---|---|
| **GatewayClass** | infrastructure provider | a type of gateway and the controller that implements it |
| **Gateway** | cluster operator | an actual entry point: listeners, ports, TLS |
| **HTTPRoute** / **GRPCRoute** | application developer | how requests on a Gateway reach backend Services |

It can express things Ingress needed vendor annotations for, such as header matching
and weighted traffic splitting.

**Controller-specific CRDs.** Traefik's `IngressRoute` is a richer, Traefik-only
alternative to Ingress. It is what this cluster uses today. It works well but ties
the routing config to Traefik.

**Tunnels.** A tunnel connector running in the cluster (for example `cloudflared`)
dials *out* to a provider's edge and forwards requests back in to a Service. No
inbound ports are opened on the router. It is a pipe, not a router: the host name
routing still happens in the ingress controller.

### 8.6 ServiceLB in detail (K3s-specific)

When a `LoadBalancer` Service is created, ServiceLB:

1. Creates a **DaemonSet in `kube-system`** for that Service, whose Pods have names
   starting with `svclb-`.
2. Each of those Pods uses a **hostPort** equal to the Service port, so the node itself
   listens on, for example, 80 and 443.
3. Incoming traffic on the host port is forwarded to the Service's ClusterIP.
4. The node IPs where those Pods run are written into the Service's
   `status.loadBalancer.ingress`, which is what `kubectl get svc` shows as the
   external IP. A node's external IP is used if it has one, otherwise its internal IP.

Controls:

- Label nodes with `svccontroller.k3s.cattle.io/enablelb=true` to switch to an
  allow-list: only labelled nodes run ServiceLB Pods.
- Use `svccontroller.k3s.cattle.io/lbpool=<pool>` on nodes and Services to split
  them into pools.
- Start K3s with `--disable=servicelb` to use something else, such as MetalLB.

Because it relies on host ports, two LoadBalancer Services cannot both use the same
port on the same node. A node where the port is already taken simply does not run
that Service's ServiceLB Pod.

### 8.7 NetworkPolicy: restricting traffic

By default every Pod accepts traffic from anywhere and can send traffic anywhere.
A **NetworkPolicy** changes that for the Pods it selects:

- A Pod becomes **isolated for ingress** when any policy selecting it lists `Ingress`
  in `policyTypes`. From then on only traffic that some policy allows gets in.
- The same applies to **egress** with `Egress`.
- **Policies add up.** There are no deny rules; if any policy allows a connection, it
  is allowed.
- **Both ends must allow it.** The sender's egress and the receiver's ingress both
  need to permit the connection.
- Traffic to and from the Pod's own node is always allowed.
- Policies only work if the network plugin enforces them. K3s ships an embedded
  network policy controller, which can be turned off with `--disable-network-policy`.

A common starting point is to deny all ingress in a namespace, then allow what is
needed:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: web
spec:
  podSelector: {}          # every Pod in the namespace
  policyTypes: ["Ingress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-traefik
  namespace: web
spec:
  podSelector:
    matchLabels: { app: web }
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: TCP
          port: 8080
```

NetworkPolicy works on IPs and ports (layers 3 and 4). It cannot allow "github.com
only", because domain names map to changing IPs. That needs an egress proxy.

### 8.8 Ports K3s needs between nodes

From the K3s requirements page:

| Protocol | Port | From | To | Purpose |
|---|---|---|---|---|
| TCP | 2379-2380 | servers | servers | embedded etcd (HA only) |
| TCP | 6443 | agents and nodes | servers | Kubernetes API |
| UDP | 8472 | all nodes | all nodes | flannel VXLAN |
| TCP | 10250 | all nodes | all nodes | kubelet metrics and API |
| UDP | 51820 / 51821 | all nodes | all nodes | flannel WireGuard (IPv4 / IPv6), only with that backend |
| TCP | 5001 | all nodes | all nodes | embedded distributed registry, only if enabled |

A host firewall that silently blocks 2380 between servers breaks etcd while
everything else looks fine. That was one of the two faults in the September outage.

### 8.9 How this cluster puts it together

- **Public traffic** (`*.packetcraft.dev`): Cloudflare's edge terminates TLS and sends
  requests down a Cloudflare Tunnel to `cloudflared`, which runs as a Deployment inside
  the cluster and forwards to the Traefik Service. Traefik routes by host name with
  IngressRoutes.
- **Private traffic** (`*.lab.packetcraft.dev`): DNS points at private LAN addresses.
  Off the LAN, those are reached over Tailscale through a subnet router. Traefik is a
  LoadBalancer Service served by ServiceLB, so it answers on ports 80 and 443 of every
  node, and kube-vip also holds a shared lab address.
- **The API server** is reached through the kube-vip address `192.168.1.241`, which
  moves between the three servers.

---

## 9. Storage: PVCs, StorageClasses and node locality

The flow:

- A **StorageClass** describes how to create volumes, and which provisioner does it.
- A Pod asks for storage with a **PVC** ("10Gi, ReadWriteOnce").
- The provisioner creates a **PV** to satisfy it, and the PVC binds to it.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: web
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
```

```yaml
# in the pod spec
    volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data
    containers:
      - name: app
        volumeMounts:
          - { name: data, mountPath: /var/lib/app }
```

**Access modes:**

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | read-write from a single **node** at a time |
| `ReadOnlyMany` (ROX) | read-only from many nodes |
| `ReadWriteMany` (RWX) | read-write from many nodes; needs a network filesystem or Longhorn RWX |
| `ReadWriteOncePod` (RWOP) | read-write from a single **Pod** |

**StorageClass settings that matter:**

- `reclaimPolicy`: `Delete` removes the underlying volume when the PVC is deleted;
  `Retain` keeps it for manual recovery.
- `volumeBindingMode: WaitForFirstConsumer` delays creating the volume until a Pod is
  scheduled, so the volume is created on a node that Pod can actually use. K3s's
  `local-path` class works this way.
- `allowVolumeExpansion: true` lets you grow a PVC by editing its size.

**The two classes on this cluster:**

- **`local-path`** (bundled with K3s): a directory on the node where the Pod first
  landed. Fast and simple, but **the data is pinned to that node**. If the Pod moves,
  the data does not follow.
- **Longhorn**: replicated block storage across nodes with dedicated disks, with
  snapshots and backups. Pods can move between nodes and keep their data. Here it runs
  with two replicas (nebula and legion), because those are the only two nodes with
  Longhorn disks.

Replicated storage is not a backup. It protects against losing a disk or a node, not
against deleting the data or corrupting it. Backups go to a separate target, and a
backup only counts once you have restored from it.

---

## 10. Configuration and secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config, namespace: web }
data:
  UPSTREAM_HOST: "backend.web.svc.cluster.local"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata: { name: app-secret, namespace: web }
type: Opaque
stringData:                 # plain text in, stored base64 encoded
  api_token: "s3cr3t"
```

```yaml
    containers:
      - name: app
        envFrom:
          - configMapRef: { name: app-config }
        env:
          - name: API_TOKEN
            valueFrom:
              secretKeyRef: { name: app-secret, key: api_token }
```

**Secrets are base64 encoded, not encrypted.** Anyone who can read Secrets in a
namespace can decode them. Two separate problems need solving:

- **In Git:** plain Secrets cannot be committed. Sealed Secrets, SOPS or External
  Secrets Operator solve this. This cluster uses Sealed Secrets, where only the
  in-cluster controller can decrypt, and the controller's key must be backed up
  outside the cluster.
- **At rest in the datastore:** K3s can encrypt Secrets in etcd with
  `--secrets-encryption`, which is off by default.

Environment variables from a Secret are read when the container starts. Changing the
Secret does not update a running container's environment; restart the Deployment.
Values mounted as files are updated eventually.

---

## 11. Health checks: probes

```yaml
    containers:
      - name: app
        livenessProbe:        # "is it alive?"  failing: restart the container
          httpGet: { path: /healthz, port: 8080 }
          periodSeconds: 10
        readinessProbe:       # "ready for traffic?"  failing: removed from Service endpoints
          httpGet: { path: /ready, port: 8080 }
          periodSeconds: 5
        startupProbe:         # "still starting?"  holds off the other probes
          httpGet: { path: /healthz, port: 8080 }
          failureThreshold: 30
          periodSeconds: 2
```

A **liveness** failure restarts the container; a **readiness** failure only stops
traffic. Using liveness for a slow but healthy app causes restart loops. Liveness
probes should not check dependencies such as the database, or one database hiccup
restarts every Pod at once. Put dependency checks in readiness.

---

## 12. Security basics

### RBAC

Access to the API is controlled with four objects:

| Object | Scope | Purpose |
|---|---|---|
| **Role** | one namespace | a set of allowed verbs on resources |
| **ClusterRole** | cluster-wide | same, for cluster-scoped resources or for reuse across namespaces |
| **RoleBinding** | one namespace | grants a Role (or a ClusterRole, limited to that namespace) to subjects |
| **ClusterRoleBinding** | cluster-wide | grants a ClusterRole everywhere |

Permissions only add up; **there are no deny rules**. Subjects are users, groups or
**ServiceAccounts**, which is how workloads get API access.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: web
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: web
subjects:
  - kind: ServiceAccount
    name: dashboard
    namespace: web
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Every Pod runs as a ServiceAccount (`default` if none is set). Give workloads their
own ServiceAccount with only what they need, and set
`automountServiceAccountToken: false` for Pods that never call the API. Binding a
dashboard to `cluster-admin` is convenient and gives anyone who reaches that dashboard
full control of the cluster.

### Pod Security Admission

Kubernetes has three built-in Pod Security Standards, enforced per namespace with
labels:

- **privileged**: no restrictions.
- **baseline**: blocks known privilege escalations (host namespaces, privileged
  containers and similar).
- **restricted**: current hardening best practice (non-root, dropped capabilities,
  seccomp).

Each level can be applied in three modes: `enforce` rejects violating Pods, `audit`
records them in the audit log, `warn` shows a warning to the user.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
```

### securityContext

The per-Pod and per-container settings that most workloads should have:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities: { drop: ["ALL"] }
```

Note that `readOnlyRootFilesystem` is a **container** field. Putting it in the Pod
level `securityContext` is invalid, which server-side apply will reject.

---

## 13. Helm, Kustomize and K3s's Helm integration

- **Kustomize** patches plain YAML with overlays and an `images:` block, no templating
  language. Built into `kubectl apply -k`. Good for your own manifests.
- **Helm** packages templated, parameterised charts installed with `values.yaml`.
  Good for other people's complex software.

Rule of thumb: **Helm to install other people's software, Kustomize to manage your
own manifests.** ArgoCD understands both.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prom prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
helm list -A
helm get values kube-prom -n monitoring
```

**K3s-specific:** K3s has its own Helm controller. A `HelmChart` resource installs a
chart, and a `HelmChartConfig` resource overrides values for a chart K3s already
installs, such as its bundled Traefik. This is how you change Traefik's settings
without disabling the packaged component.

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    deployment:
      replicas: 2
```

---

## 14. Deployment modes

### 14.1 Ways to run Kubernetes in general

| Approach | Examples | You manage |
|---|---|---|
| Managed service | EKS, GKE, AKS | worker nodes and workloads; the provider runs the control plane |
| Self-managed upstream | kubeadm | everything, including etcd and upgrades |
| Lightweight distribution | K3s, k0s, MicroK8s | the nodes; the distribution simplifies the control plane |
| Local development | kind, minikube | a throwaway cluster on one machine |

For self-managed HA clusters there are two etcd layouts:

- **Stacked etcd**: etcd runs on the control-plane nodes themselves. Fewer machines,
  but losing a node loses both a control-plane instance and an etcd member. This is
  what K3s embedded etcd is.
- **External etcd**: etcd runs on its own hosts. More machines, but control-plane and
  datastore failures are separate.

### 14.2 K3s cluster shapes

**Single server with SQLite (the default).**

```bash
curl -sfL https://get.k3s.io | sh -
```

One server with an embedded SQLite datastore, plus any number of agents. Simple and
light, but the server is a single point of failure, and **SQLite cannot be used with
more than one server.** Nothing tells you this is what you are running; check for
`/var/lib/rancher/k3s/server/db/state.db`.

**High availability with embedded etcd.**

```bash
# first server
curl -sfL https://get.k3s.io | K3S_TOKEN=<secret> sh -s - server --cluster-init \
  --tls-san 192.168.1.241

# each additional server
curl -sfL https://get.k3s.io | K3S_TOKEN=<secret> sh -s - server \
  --server https://<first-server>:6443 --tls-san 192.168.1.241
```

- Needs an **odd number of servers**, three or more, so etcd can keep quorum. Three
  servers tolerate one failure; five tolerate two. Two servers tolerate none.
- All servers must use the same networking and feature flags.
- An existing single-server SQLite cluster can be converted by restarting its server
  with `--cluster-init`.
- This is how this cluster runs (deus, sol, opus).

**High availability with an external datastore.**

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="postgres://user:pass@db-host:5432/k3s"
```

- Two or more servers share an external MySQL, MariaDB, PostgreSQL or etcd.
- The K3s docs suggest this if your team knows how to run a SQL database but not
  etcd. The datastore then needs its own HA and backups.
- Pass credentials through the `K3S_DATASTORE_ENDPOINT` environment variable rather
  than the command line, so they do not show up in process listings.

**Agents.**

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.241:6443 K3S_TOKEN=<token> sh -
```

Point agents at a **fixed registration address** (a VIP or DNS name) rather than one
server, so a new agent can still join when that server is down. After registering, the
agent connects to all servers through its client-side load balancer.

**Dedicated control-plane servers.** To keep ordinary workloads off servers, start
them with a taint such as `--node-taint CriticalAddonsOnly=true:NoExecute`. Only Pods
that tolerate it (system components) will run there.

### 14.3 Choosing

| Situation | Shape |
|---|---|
| Learning, CI, a single box | single server, SQLite |
| Small cluster that must survive losing a server | three servers, embedded etcd |
| You already run a reliable SQL database | two or more servers, external datastore |
| Control plane on weak hardware or doing critical work | tainted dedicated servers plus agents |

---

## 15. K3s operations

### Configuration

Everything that can be a flag can go in `/etc/rancher/k3s/config.yaml`, which is
easier to manage with Ansible than editing the systemd unit:

```yaml
# /etc/rancher/k3s/config.yaml on a server
tls-san:
  - 192.168.1.241
write-kubeconfig-mode: "0644"
etcd-expose-metrics: true
disable:
  - servicelb        # only if replacing it
```

- `tls-san` adds names and IPs to the API certificate. Every address you use to reach
  the API must be listed.
- The kubeconfig at `/etc/rancher/k3s/k3s.yaml` is written with mode 600 by default.
- `etcd-expose-metrics` is **false** by default, so etcd is invisible to Prometheus
  until you turn it on.

### Services and logs

```bash
sudo systemctl status k3s          # on a server
sudo journalctl -u k3s -f
sudo systemctl status k3s-agent    # on an agent
sudo journalctl -u k3s-agent -f
sudo cat /var/lib/rancher/k3s/server/node-token   # join token
sudo k3s crictl ps                 # containers at the runtime level
sudo k3s crictl images
```

### Packaged components and auto-deploying manifests

K3s deploys CoreDNS, Traefik, local-storage and metrics-server from manifests in
`/var/lib/rancher/k3s/server/manifests/`, plus the embedded ServiceLB controller. Any
file you place in that directory is applied like `kubectl apply`, at startup and
whenever the file changes.

- **Deleting a file there does not delete what it created.**
- To remove or skip a packaged component, start K3s with `--disable=<name>` (for
  example `--disable=traefik`). Deleting its manifest just gets it recreated.
- A `<name>.yaml.skip` file stops K3s deploying that manifest, but does not remove an
  add-on that already exists.

### etcd snapshots and restore

With embedded etcd, K3s takes snapshots automatically:

| Setting | Default |
|---|---|
| Schedule | 00:00 and 12:00 system time |
| Retention (`--etcd-snapshot-retention`) | 5 snapshots |
| Location (`--etcd-snapshot-dir`) | `/var/lib/rancher/k3s/server/db/snapshots` |

Snapshots can also go to S3-compatible storage with `--etcd-s3` plus
`--etcd-s3-endpoint`, `--etcd-s3-bucket`, `--etcd-s3-access-key` and
`--etcd-s3-secret-key`. Keeping them only on the servers means losing the servers
loses the snapshots too.

```bash
sudo k3s etcd-snapshot save          # take one now
sudo k3s etcd-snapshot ls
```

Restoring an HA cluster, from the K3s docs:

1. Stop K3s on **all** servers.
2. On the first server:
   `k3s server --cluster-reset --cluster-reset-restore-path=<snapshot>`
3. On every other server, delete `/var/lib/rancher/k3s/server/db/`.
4. Start K3s on the first server, then the others. They rejoin the restored cluster.

Restoring onto new hardware also needs the original **server token**, because it
decrypts bootstrap data inside the snapshot. Back up the token with the snapshots.

### Upgrades

- **Manual:** re-run the install script with the version you want
  (`INSTALL_K3S_VERSION=<version>`), one node at a time, **servers before agents**.
- **Automated:** the system-upgrade-controller reads `Plan` resources that say which
  nodes to upgrade and to what version (or which release channel to follow). A
  `concurrency` field limits how many nodes upgrade at once. The agent plan waits for
  the server plan to finish.

Take an etcd snapshot before any upgrade, and never skip minor versions.

### Node lifecycle

```bash
kubectl cordon <node>        # no new Pods; existing ones stay
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data   # evict Pods (respects PDBs)
kubectl uncordon <node>      # back into rotation
```

`drain` uses the Eviction API, so PodDisruptionBudgets are honoured. DaemonSet Pods
are skipped because they belong to the node. **kured** automates this for OS updates:
it watches for the reboot-required flag, then cordons, drains, reboots and uncordons
one node at a time.

**On a server node,** draining only moves workloads. The etcd member on that node
still goes down with the reboot, so with three servers the cluster has no margin left
until it returns. Check that the other servers are healthy first.

---

## 16. Observability basics

- **metrics-server** (bundled with K3s) provides the resource metrics API behind
  `kubectl top` and CPU or memory based HPAs. It keeps only recent values; it is not a
  monitoring system.
- **Prometheus**, usually through kube-prometheus-stack, scrapes node-exporter,
  kube-state-metrics, the kubelet and anything with a ServiceMonitor or PodMonitor.
- **K3s blind spots:** the upstream chart expects etcd, the scheduler and the
  controller manager to be separate Pods. In K3s they are inside the `k3s` process, so
  those scrape jobs can have zero targets without any error. Turn on
  `etcd-expose-metrics`, point scrape configs at the servers' IPs, and confirm targets
  are actually `up` in Prometheus.
- **Test your alerts** by causing the condition. A rule that loads is not a rule that
  fires.

Alerts worth having on a small cluster from day one:

```
up == 0                                     # a scrape target disappeared
changes(node_boot_time_seconds[1h]) > 0     # a node rebooted
kube_node_status_condition{condition="Ready",status="true"} == 0
etcd_server_has_leader == 0                 # needs etcd metrics enabled
```

---

## 17. A debugging playbook

```bash
# Pod will not start: read the events FIRST
kubectl describe pod <pod> -n <ns>
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

| State | Meaning | First move |
|---|---|---|
| `Pending` | the scheduler cannot place it | `describe pod`: no node matches selectors or taints, not enough resources, or a PVC is unbound |
| `ImagePullBackOff` | cannot pull the image | wrong name or tag, uppercase in a ghcr.io path, or a private registry needs a pull secret |
| `CrashLoopBackOff` | starts then dies repeatedly | `kubectl logs <pod> --previous` |
| `CreateContainerConfigError` | a referenced ConfigMap or Secret key is missing | check names and keys, and the namespace |
| `ContainerCreating` (stuck) | volume will not attach or mount | `describe pod` for the mount error; check the storage system |
| `Running` but no traffic | readiness failing, or a selector mismatch | check the Service's EndpointSlices |
| `Terminating` (stuck) | finalizers, or the node is gone | `kubectl get <kind> <name> -o yaml` and look at `finalizers` |
| Node `NotReady` | kubelet cannot reach the API | on the node: `journalctl -u k3s` (or `k3s-agent`), then check the network |

### Network debugging

```bash
# Does the Service have backends?
kubectl get endpointslices -n <ns> -l kubernetes.io/service-name=<svc>
kubectl get pods -n <ns> --show-labels      # compare with the Service selector

# Test from inside the cluster
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh
#   nslookup web.web.svc.cluster.local
#   wget -qO- http://web.web.svc.cluster.local

# Is CoreDNS healthy?
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Is a NetworkPolicy in the way?
kubectl get networkpolicy -A
```

On a node, when something can send traffic but not receive replies, look at routing
rather than the application: `ip rule`, `ip route get <peer>`, and
`ip route show table all`.

### Control plane health

```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl top nodes
kubectl top pods -A
# from a server, check etcd peers are reachable
nc -vz <other-server> 2380
```

---

## 18. Common workload shapes

- **Stateless service:** Deployment with two or more replicas, a ClusterIP Service,
  an Ingress or IngressRoute, a PDB, and topology spread across nodes.
- **Database:** an operator (CloudNativePG) with replicated storage (Longhorn),
  continuous backups to object storage, and a tested restore.
- **GPU workload:** a node label for attraction, a toleration if the node is tainted,
  and an `nvidia.com/gpu: 1` resource request. The NVIDIA device plugin DaemonSet must
  be installed for that resource to exist at all.
- **Node agent:** DaemonSet with tolerations for every taint in the cluster, so it
  also runs on control-plane nodes.
- **Scheduled task:** CronJob with `concurrencyPolicy: Forbid` and a sensible
  `backoffLimit`.

---

## 19. Minimal end-to-end example

Do this by hand once. It shows what the automation does for you later.

```bash
kubectl create namespace demo
kubectl create deployment hello --image=nginx:1.27-alpine -n demo
kubectl expose deployment hello --port=80 -n demo        # ClusterIP Service
kubectl get pods,svc -n demo -o wide
kubectl scale deployment/hello -n demo --replicas=3
kubectl get pods -n demo -o wide -w                      # spread across nodes?
kubectl get endpointslices -n demo                       # three endpoints

kubectl port-forward -n demo svc/hello 8080:80
# curl localhost:8080 shows the nginx welcome page

kubectl delete namespace demo                            # removes everything inside
```

Then notice that a GitOps loop does all of this from a `git push`: declaratively,
self-healing, and pinned to an exact image build.

---

## Quick reference card

```bash
# context and cluster
kubectl get nodes -o wide
kubectl config current-context
kubectl api-resources

# the daily four
kubectl get pods -A
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> -f
kubectl exec -it <pod> -n <ns> -- sh

# apply and rollout
kubectl apply -k apps/<svc>/
kubectl rollout status deployment/<svc> -n <ns>
kubectl rollout undo deployment/<svc> -n <ns>

# scheduling
kubectl label node <node> <key>=<val>
kubectl taint node <node> <key>=<val>:NoSchedule
kubectl cordon <node> ; kubectl drain <node> --ignore-daemonsets ; kubectl uncordon <node>

# networking
kubectl get svc,endpointslices -n <ns>
kubectl get networkpolicy -A

# debug
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl top nodes ; kubectl top pods -A

# K3s on the node
sudo journalctl -u k3s -f
sudo k3s etcd-snapshot save
sudo k3s crictl ps
```

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
