# Objects and Workloads

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

The objects that run your applications, how a manifest is put together, and the workload features (rollouts, sidecars, autoscaling, disruption budgets, probes) you will use day to day.

## The object hierarchy: what actually runs your stuff

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
  Covered in depth on the [networking](networking.md) page.
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

## Anatomy of a manifest

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

## Workload resources in more depth

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

## Health checks: probes

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

## Sources

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
