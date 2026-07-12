# K3s / Kubernetes — Working Reference

A practical guide to Kubernetes, oriented around K3s. It's written from the seat
of someone running a small heterogeneous cluster — one always-on control-plane
node plus a couple of workers of differing capability — so where a real cluster
made a concept concrete, I've kept that context in. Everything here is standard
Kubernetes unless a section is explicitly flagged K3s-specific.

Throughout, I refer to my own three nodes by name where it helps:
- **the control-plane node** — a small always-on mini PC; runs the K3s server
  and anything stateful-ish that must never move.
- **a GPU worker** — a laptop-class node with a discrete GPU; transient, so I
  taint it.
- **a storage worker** — a desktop with the disks; where storage-bound work
  lands.

Substitute your own node names freely.

---

## 0. K3s vs Kubernetes — settle it first

K3s **is** Kubernetes. It's a CNCF-certified conformant distribution, so every
API object, every `kubectl` verb, every manifest you write works identically on
upstream Kubernetes, EKS, GKE, or a 500-node cluster. What K3s changes is
*packaging and operations*, not the API:

| Concern | Upstream k8s | K3s |
|---|---|---|
| Install | kubeadm + many components | one `curl \| sh` binary |
| Datastore | external etcd cluster | SQLite (single server) or embedded etcd (HA) |
| CNI (networking) | you pick & install | Flannel, bundled |
| Ingress controller | you install | Traefik, bundled |
| Storage class | you install | `local-path`, bundled (default) |
| Service LB | cloud provider | Klipper / ServiceLB, bundled |
| Binary footprint | large | ~70MB, runs on a Pi |

**Takeaway:** describe it as "K3s, a conformant Kubernetes distribution" — it
signals you understand the distinction. The skills transfer directly; there's no
"just homelab" asterisk on them.

---

## 1. The mental model

Kubernetes is a **reconciliation engine**. You declare desired state (YAML
manifests); controllers continuously work to make actual state match. You almost
never tell it to *do* a thing — you tell it what should *be true*, and it figures
out the actions. If you run GitOps on top (e.g. ArgoCD's `selfHeal`), that's the
same principle one layer up: the GitOps controller reconciles Git → cluster,
while the Kubernetes control plane reconciles manifests → running containers.

The flow when you `kubectl apply` a Deployment:

```
You ──apply──▶ API server ──writes──▶ datastore (SQLite/etcd on the control-plane node)
                    │
                    ▼ (controllers watch the datastore)
        Deployment controller: "I need 1 ReplicaSet"
        ReplicaSet controller: "I need 1 Pod"
        Scheduler:             "that Pod goes on... a worker"
                    │
                    ▼
        kubelet on that worker: "pulling image, starting container"
```

Nothing pushes to nodes. Every node's **kubelet** *pulls* its assigned work from
the API server. It's a pull model end to end.

---

## 2. The object hierarchy (what actually runs your stuff)

From smallest to largest, the chain that turns a container image into a running
thing:

- **Container** — your image (`registry/app:tag`), running.
- **Pod** — one or more containers sharing a network namespace + storage. The
  *smallest schedulable unit*. Pods are cattle: disposable, replaceable, get a
  new IP every time. You rarely create these directly.
- **ReplicaSet** — keeps N identical Pods alive. You rarely touch these directly.
- **Deployment** — manages ReplicaSets to give you rolling updates and rollbacks.
  **This is what you write** for stateless services (web servers, APIs, proxies).
- **StatefulSet** — like a Deployment but Pods get stable identities and stable
  storage (`pod-0`, `pod-1`). For databases and anything where identity matters.
- **DaemonSet** — one Pod on *every* node (or every matching node). Used for
  node-level agents: Prometheus `node-exporter`, log collectors, CNI, device
  plugins.
- **Job / CronJob** — run-to-completion and scheduled tasks. Backups, batch,
  migrations.

Supporting objects:

- **Service** — a stable virtual IP + DNS name fronting a set of Pods (Pods die;
  the Service endpoint doesn't). Three flavors in §6.
- **Namespace** — a logical partition (`argocd`, `monitoring`, `kube-system`,
  one per app). Scopes names and lets you apply quotas/policies per tenant.
- **ConfigMap / Secret** — config and credentials injected into Pods as env vars
  or mounted files. Secrets are base64, *not* encrypted at rest by default.
- **PersistentVolume (PV) / PersistentVolumeClaim (PVC)** — storage abstraction.
  A Pod asks for storage via a PVC; a StorageClass provisions a PV to satisfy it.
- **Ingress / IngressRoute** — HTTP routing from outside into Services. With
  Traefik you get its `IngressRoute` CRD, a richer Traefik-specific version of
  the standard `Ingress` object.

---

## 3. kubectl — the commands you'll actually run

`kubectl` is your single interface. The pattern is almost always
`kubectl <verb> <resource> [name] [flags]`.

### Inspection (90% of daily use)
```bash
kubectl get nodes -o wide              # all nodes, IPs, k3s version, OS
kubectl get pods -A                    # -A = all namespaces (kube-system too)
kubectl get pods -n <ns> -o wide       # which node each pod landed on
kubectl get all -n <ns>                # pods, services, deployments, etc. in one ns
kubectl get deployment,svc,pvc -n <ns>

kubectl describe pod <pod> -n <ns>     # events, why it won't schedule, image pull errs
kubectl describe node <node>           # capacity, taints, what's scheduled there

kubectl logs <pod> -n <ns>             # stdout/stderr
kubectl logs <pod> -n <ns> -f          # follow (tail -f)
kubectl logs <pod> -n <ns> --previous  # logs from the crashed prior container
```

### Interacting with running Pods
```bash
kubectl exec -it <pod> -n <ns> -- sh   # shell inside a container
kubectl exec <pod> -n <ns> -- nginx -t          # one-off command
kubectl cp <pod>:/etc/nginx/nginx.conf ./out.conf -n <ns>
```

### Port-forwarding (a common gotcha)
```bash
kubectl port-forward -n <ns> svc/<svc> 8888:80
```
If you run this **inside an SSH session** to a node, `localhost:8888` binds to
*that node's* loopback, not your laptop. Two fixes:
```bash
ssh -L 8888:localhost:8888 <node>      # tunnel the port back to your laptop, OR
kubectl port-forward --address 0.0.0.0 -n <ns> svc/<svc> 8888:80  # then hit the node's LAN IP
```
The clean long-term fix: copy the kubeconfig (`/etc/rancher/k3s/k3s.yaml` on a
K3s server) to your workstation, rewrite the `server:` field to the control-plane
node's IP (`https://<control-plane-ip>:6443`), and run `kubectl` locally. Then
`localhost` means your laptop and forwards Just Work.

### Applying & deleting
```bash
kubectl apply -f deployment.yaml       # declarative — create or update to match file
kubectl apply -k apps/<svc>/           # apply a Kustomize directory
kubectl delete -f deployment.yaml
kubectl delete pod <pod> -n <ns>       # pod gets recreated by its ReplicaSet — a quick restart
```
**Prefer `apply` (declarative) over `create`/`edit` (imperative).** Everything in
the cluster should be reproducible from source. The imperative commands are for
debugging and learning, not for the running state — and if you run GitOps, a
`selfHeal` controller will literally undo any imperative change you make to a
managed resource, by design.

### Rollouts (Deployments)
```bash
kubectl rollout status deployment/<svc> -n <ns>
kubectl rollout history deployment/<svc> -n <ns>
kubectl rollout undo deployment/<svc> -n <ns>          # roll back one version
kubectl rollout restart deployment/<svc> -n <ns>       # restart all pods (re-pull, re-read config)
```

### Scaling
```bash
kubectl scale deployment/<svc> -n <ns> --replicas=3
# (a GitOps controller reverts this if the app is git-managed — the self-heal proof)
```

### Quality-of-life
```bash
kubectl get pods -A -w                 # -w = watch, live updates
kubectl get events -n <ns> --sort-by=.lastTimestamp   # what just happened & why
kubectl api-resources                  # every object type + its short name & API group
kubectl explain deployment.spec.template.spec.containers  # inline schema docs
alias k=kubectl                        # everyone does this
```

---

## 4. Anatomy of a manifest

Every object shares the same four top-level keys:

```yaml
apiVersion: apps/v1     # which API group+version owns this kind
kind: Deployment        # the object type
metadata:
  name: web             # unique within namespace+kind
  namespace: web
spec:                   # DESIRED STATE — the only part you really design
  replicas: 1
  selector:             # how this Deployment finds "its" pods (must match template labels)
    matchLabels: { app: web }
  template:             # the Pod blueprint stamped out N times
    metadata:
      labels: { app: web }   # ← selector matches THIS
    spec:
      nodeSelector:
        kubernetes.io/hostname: <control-plane-node>   # placement constraint (see §5)
      containers:
        - name: web
          image: registry/web-app:latest
          ports:
            - containerPort: 80
# status:               # ACTUAL STATE — k8s writes this, you never do
```

The **labels/selector** coupling trips everyone up early: a Service finds Pods by
label selector, a Deployment manages Pods by label selector. If labels and
selectors don't match, you get a Service with no endpoints or a Deployment that
manages nothing. They're the glue.

---

## 5. Scheduling — placement in a heterogeneous cluster

This matters most when your nodes are *not* interchangeable — different CPUs,
some with a GPU, some always-on and some transient. The scheduler decides
placement **once, at creation time**, based on `resources.requests` (declared,
not actual usage) — there's no automatic rebalancing afterward. You steer it with
four tools.

### nodeSelector — simplest, exact-match
```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: <control-plane-node>    # "only here"
```
Good for "this must run on the always-on node" — I pin the GitOps controller and
any always-up service this way.

### Node labels — your own taxonomy
```bash
kubectl label node <storage-node> role=storage
kubectl label node <gpu-node> gpu=true
kubectl get nodes --show-labels
```
Then `nodeSelector: { role: storage }` is more meaningful than a hostname and
survives a hardware swap.

### Affinity / anti-affinity — expressive rules
`nodeSelector` is hard AND-only. Affinity adds OR, preferences (soft), and
pod-relative rules:
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # hard requirement
        nodeSelectorTerms:
          - matchExpressions:
              - key: gpu
                operator: In
                values: ["true"]
    podAntiAffinity:                                     # spread replicas across nodes
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels: { app: web }
            topologyKey: kubernetes.io/hostname
```

### Taints & tolerations — the node says NO by default
A **taint** repels Pods *unless* they explicitly **tolerate** it. This is exactly
the pattern for a transient node: taint it so stateful/general workloads stay
off, and only the workloads that opt in land there.
```bash
# on a transient node:
kubectl taint node <gpu-node> transient=true:NoSchedule
```
```yaml
# only a pod with this toleration can be placed there:
spec:
  tolerations:
    - key: transient
      operator: Equal
      value: "true"
      effect: NoSchedule
```
Effects: `NoSchedule` (won't place new), `PreferNoSchedule` (soft), `NoExecute`
(also *evicts* running pods that don't tolerate — this is what drains a node).

**The combined pattern for GPU routing:** label the GPU node `gpu=true`, and
taint it `transient=true:NoSchedule`. A GPU workload then needs *both* a
`nodeSelector`/affinity for `gpu=true` (to be attracted) *and* a toleration for
`transient` (to be permitted). Attraction + permission — you need both, and
that's the bit people forget.

### Resource requests & limits — feed the scheduler real numbers
```yaml
containers:
  - name: web
    resources:
      requests:           # scheduler reserves this; used for placement math
        cpu: 100m         # 100 millicores = 0.1 core
        memory: 128Mi
      limits:             # hard ceiling; CPU throttled, memory OOM-killed if exceeded
        cpu: 500m
        memory: 256Mi
```
Without requests, the scheduler assumes ~0 and can overpack a node. On modest
nodes (say 16GB) this bites fast — set requests so placement reflects reality.

---

## 6. Networking — how traffic actually flows

Three Service types, each a different exposure level:

- **ClusterIP** (default) — internal-only virtual IP. Reachable as
  `<svc>.<ns>.svc.cluster.local` from any pod. Most services should be ClusterIP
  and get exposed via Ingress.
- **NodePort** — opens a high port (30000–32767) on *every* node's IP. Crude;
  mostly a debugging/legacy step. Usually skipped.
- **LoadBalancer** — asks for an external IP. In cloud, provisions a cloud LB; in
  K3s, the bundled ServiceLB (Klipper) fakes it using node IPs. MetalLB is the
  common bare-metal alternative — but if you front the cluster with a tunnel
  (below), you may not need LoadBalancer Services or MetalLB at all.

**A tunnel-based ingress path** (the one I run) avoids opening any router ports:
```
Internet ─▶ Cloudflare edge (TLS) ─▶ cloudflared tunnel ─▶ Traefik :80
            ─▶ IngressRoute (Host match) ─▶ ClusterIP Service ─▶ Pod
```
No router ports opened, no MetalLB, no LoadBalancer Services. `cloudflared` is a
dumb pipe; Traefik is the routing brain. Every new external service = one DNS
record + one IngressRoute. It's one of several valid patterns — a LoadBalancer +
MetalLB + real Ingress is equally legitimate; the tunnel just trades a dependency
on Cloudflare for not touching the router.

**DNS inside the cluster:** CoreDNS (in `kube-system`) gives every Service a name.
Pattern: `<service>.<namespace>.svc.cluster.local`. From a pod in the same
namespace, just `<service>` works. This is how one service finds another
(e.g. a reverse proxy finding its upstreams).

**The CNI (Flannel in K3s):** gives every Pod a routable IP on an overlay network
so any pod can reach any pod across nodes without NAT. You rarely touch it; just
know the pod network (`10.42.0.0/16` by default in K3s) is separate from your LAN
(e.g. `192.168.1.0/24`).

---

## 7. Storage — PVCs, StorageClasses, and node-locality

The PVC/PV/StorageClass dance:

- A **StorageClass** is a *provisioner* — a recipe for making volumes on demand.
- A Pod declares a **PVC** ("I want 10Gi, ReadWriteOnce").
- The StorageClass dynamically creates a **PV** to satisfy it; the PVC binds.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: web
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-path        # K3s default, lives on the node's disk
  resources:
    requests:
      storage: 5Gi
```
```yaml
# in the pod spec:
    volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data
    containers:
      - name: app
        volumeMounts:
          - { name: data, mountPath: /var/lib/app }
```

**Access modes matter for a multi-node cluster:**
- `ReadWriteOnce` (RWO) — one *node* at a time. Fine for single-replica apps.
- `ReadWriteMany` (RWX) — many nodes simultaneously. Needs a networked backend
  (NFS, or Longhorn with RWX). Required for anything that must be writable from
  pods on different nodes at once.

Two common classes:
- **`local-path`** (bundled, default): writes to a directory on whatever node the
  pod lands on. Fast, zero-config, but the data is *pinned to that node* — if the
  pod reschedules elsewhere, the data doesn't follow. Fine for state that lives
  on an always-on node. **Not** fine for anything that might move.
- **Longhorn** (or similar distributed block storage): replicated across nodes,
  survives node loss, supports RWX, gives you snapshots. The "real" answer for
  stateful workloads — at the cost of needing dedicated disks and some operational
  care.

This is the crux of a **"durable data lives outside the disposable layer"**
discipline: K3s-native `local-path` is convenient but node-fragile, so
irreplaceable data belongs on a proper storage tier (a NAS/ZFS box over the
network, or replicated Longhorn) rather than on whatever node a pod happened to
land on. It's often reasonable to keep a database *outside* the cluster entirely
until you have real replicated storage in place.

---

## 8. Configuration & secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config, namespace: web }
data:
  UPSTREAM_HOST: "backend.web.svc.cluster.local"
  log_level: "info"
---
apiVersion: v1
kind: Secret
metadata: { name: app-secret, namespace: web }
type: Opaque
stringData:                 # stringData = plaintext in, base64 stored
  api_token: "s3cr3t"
```
Inject into a pod:
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
**Secrets are base64, not encrypted.** Anyone with `kubectl get secret -o yaml`
can decode them. That may be acceptable behind a private network, but the "real"
answer — Sealed Secrets, SOPS, or external-secrets — lets you commit *encrypted*
secrets to a config repo, which plain Secrets can't safely do. If you're doing
GitOps, this is the piece that makes secrets fit the model instead of being a
manual out-of-band step.

---

## 9. Health & self-healing — the probes

Kubernetes restarts and reroutes based on three probes you define per container:

```yaml
    containers:
      - name: app
        livenessProbe:        # "is it alive?" fail ⇒ kill & restart the container
          httpGet: { path: /healthz, port: 80 }
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:       # "ready for traffic?" fail ⇒ pull from Service endpoints
          httpGet: { path: /ready, port: 80 }
          periodSeconds: 5
        startupProbe:         # "still booting?" protects slow-starters from liveness
          httpGet: { path: /healthz, port: 80 }
          failureThreshold: 30
          periodSeconds: 2
```
The distinction matters: a **liveness** failure restarts the pod; a **readiness**
failure just stops sending it traffic (no restart). Misusing liveness for a
slow-but-healthy app causes restart loops. Any service you write yourself should
expose a `/healthz` (and ideally a separate `/ready`) so Kubernetes can manage it
properly.

---

## 10. Helm & Kustomize

- **Kustomize:** patches/overlays plain YAML, no templating language. An
  `images:` block in a `kustomization.yaml` rewriting an image tag in place is
  pure Kustomize. Built into `kubectl` (`kubectl apply -k`). Good for "same
  manifests, small per-environment differences."
- **Helm:** a package manager. A "chart" is a templated, parameterized bundle;
  you install it with `--set` overrides or a `values.yaml`. Good for consuming
  complex off-the-shelf software (e.g. `kube-prometheus-stack`).

Rule of thumb: **Helm to install other people's complex software, Kustomize to
manage your own manifests.** A GitOps controller like ArgoCD natively understands
both, so either can sit behind the loop.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prom prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
helm list -A                    # what's installed
helm get values kube-prom -n monitoring   # what settings are active
helm upgrade kube-prom prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
```

---

## 11. K3s-specific operations

Things that are K3s-particular rather than generic Kubernetes:

```bash
# the kubeconfig (copy this to your laptop, rewrite the server: field)
sudo cat /etc/rancher/k3s/k3s.yaml

# server (control plane) service
sudo systemctl status k3s
sudo journalctl -u k3s -f                 # control-plane logs

# agent service — on a worker
sudo systemctl status k3s-agent
sudo journalctl -u k3s-agent -f

# the join token (to add a node)
sudo cat /var/lib/rancher/k3s/server/node-token

# add an agent (run on the new node)
curl -sfL https://get.k3s.io | K3S_URL=https://<control-plane-ip>:6443 \
  K3S_TOKEN=<token> sh -

# K3s bundles its own containerd + crictl for low-level container debugging
sudo k3s crictl ps                        # running containers at the runtime level
sudo k3s crictl images

# clean uninstall (K3s ships the script)
/usr/local/bin/k3s-uninstall.sh           # on a server
/usr/local/bin/k3s-agent-uninstall.sh     # on an agent
```

**Node lifecycle** (the graceful way to power a node down):
```bash
kubectl cordon <node>        # mark unschedulable — no NEW pods, existing stay
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data  # evict pods, prep for shutdown
kubectl uncordon <node>      # back into rotation
```
`drain` before powering a node down reschedules evictable pods elsewhere first —
the right move for any node you shut down regularly (a transient laptop worker, a
node you patch and reboot). DaemonSets are excluded because they're node-bound by
definition; hence `--ignore-daemonsets`.

**Bundled-component note:** because Traefik, ServiceLB, and local-path ship
*inside* K3s, they're managed by manifests in
`/var/lib/rancher/k3s/server/manifests/`. K3s re-applies these on restart. To
disable a bundled component (e.g. swap Traefik for something else), start k3s with
`--disable=traefik` rather than deleting the manifest, which would just get
recreated.

---

## 12. A debugging playbook (the 80% of incidents)

```bash
# Pod won't start — read the events FIRST, they almost always say why
kubectl describe pod <pod> -n <ns>      # look at the Events: section at the bottom
kubectl get events -n <ns> --sort-by=.lastTimestamp
```
Common states and what they mean:

| State | Meaning | First move |
|---|---|---|
| `Pending` | scheduler can't place it | `describe pod` → no node matches selector/taint, or insufficient resources |
| `ImagePullBackOff` | can't pull the image | private registry needs a pull secret, or wrong image name/case |
| `CrashLoopBackOff` | container starts then dies repeatedly | `kubectl logs <pod> --previous` to see the death |
| `ContainerCreating` (stuck) | volume won't mount / image still pulling | `describe pod` for the mount or pull error |
| `Running` but no traffic | readiness probe failing or label/selector mismatch | check `kubectl get endpoints <svc>` — empty means selector miss |
| `Terminating` (stuck) | finalizers or a wedged node | `describe`, check node health |

```bash
# Service has no endpoints? The classic label/selector bug:
kubectl get endpoints <svc> -n <ns>     # empty = Service selector doesn't match any Pod labels
kubectl get pods -n <ns> --show-labels  # compare to the Service's selector

# Is the whole control plane healthy?
kubectl get pods -n kube-system         # CoreDNS, Traefik, metrics-server all Running?

# What's eating resources? (needs metrics-server, bundled in K3s)
kubectl top nodes
kubectl top pods -A
```

---

## 13. Common workload shapes

A few recurring patterns, in increasing order of scheduling complexity:

- **Stateless service:** Deployment + ClusterIP Service + IngressRoute, pinned to
  a specific node if it must be always-on. The template for everything stateless
  — web servers, APIs, reverse proxies.
- **GPU workload:** Deployment with `nodeSelector`/affinity for a `gpu=true`
  label + a toleration for whatever taint guards that node + a GPU resource
  request (`nvidia.com/gpu: 1`, once the NVIDIA device plugin DaemonSet is
  installed). The device plugin is the missing link that exposes the GPU to the
  scheduler — without it, `nvidia.com/gpu` requests never satisfy.
- **Stateful service (database, etc.):** StatefulSet + a PVC on replicated
  storage (Longhorn/NFS), *not* `local-path`, so the data survives a
  reschedule. If you don't yet have replicated storage, it's often cleaner to run
  the database outside the cluster than to pin it fragilely to one node.
- **Node-level agent:** DaemonSet — exporters, log shippers, device plugins.

---

## 14. Minimal end-to-end example to internalize the model

Deploy something from scratch, by hand, once — then never again (because you'll
move to declarative manifests / GitOps). It cements what the automation is doing
for you:

```bash
kubectl create namespace demo

kubectl create deployment hello --image=nginx:alpine -n demo
kubectl expose deployment hello --port=80 -n demo          # makes a ClusterIP Service
kubectl get pods,svc -n demo -o wide                       # see where it landed
kubectl scale deployment/hello -n demo --replicas=3        # watch 3 pods appear
kubectl get pods -n demo -o wide -w                        # spread across nodes?

kubectl port-forward -n demo svc/hello 8080:80             # (mind the SSH-localhost gotcha, §3)
# curl localhost:8080 → nginx welcome page

kubectl delete namespace demo                              # tears down everything in it
```
Then realize a GitOps loop does all of this from a `git push` — declaratively,
self-healing, with the image pinned to an exact build. Being able to do it by
hand *and* having built the automation that makes doing it by hand unnecessary is
the whole competency.

---

## Quick reference card

```bash
# context & cluster
kubectl get nodes -o wide
kubectl cluster-info
kubectl config current-context

# the daily four
kubectl get pods -A
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> -f
kubectl exec -it <pod> -n <ns> -- sh

# apply / rollout
kubectl apply -k apps/<svc>/
kubectl rollout status deployment/<svc> -n <ns>
kubectl rollout undo deployment/<svc> -n <ns>

# scheduling
kubectl label node <node> <key>=<val>
kubectl taint node <node> <key>=<val>:NoSchedule
kubectl cordon/drain/uncordon <node>

# debug
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl get endpoints <svc> -n <ns>
kubectl top nodes ; kubectl top pods -A

# k3s host-level (on the node)
sudo journalctl -u k3s -f               # server (control plane)
sudo journalctl -u k3s-agent -f         # agent (worker)
sudo k3s crictl ps
```
