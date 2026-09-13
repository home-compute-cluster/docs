# Scheduling: Placement in a Mixed Cluster

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

How the scheduler decides where Pods go, and the tools for steering it on nodes that are not interchangeable.

The scheduler decides placement **once, when the Pod is created**, based on declared
`resources.requests` (not actual usage) and the constraints below. There is no
automatic rebalancing afterwards.

## nodeSelector: simplest, exact match

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: nebula
```

## Node labels: your own taxonomy

```bash
kubectl label node nebula role=storage
kubectl label node legion gpu=true
kubectl get nodes --show-labels
```

`nodeSelector: { role: storage }` says what you mean and survives replacing the
hardware.

## Affinity and anti-affinity

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

## Topology spread constraints

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

## Taints and tolerations: the node says no by default

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

## Resource requests and limits

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

## Sources

- [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
