# Storage: PVCs, StorageClasses and Node Locality

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

How Pods get persistent storage, and why where the data lives matters on a small cluster.

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

**The two main classes on this cluster:**

- **`local-path`** (bundled with K3s): a directory on the node where the Pod first
  landed. Fast and simple, but **the data is pinned to that node**. If the Pod moves,
  the data does not follow.
- **Longhorn**: replicated block storage across nodes with dedicated disks, with
  snapshots and backups. Pods can move between nodes and keep their data. Here it runs
  with two replicas (nebula and legion), because those are the only two nodes with
  Longhorn disks.

Two narrower classes exist for single workloads. `nebula-local` has no provisioner
and binds Seafile to a hand-made PersistentVolume on nebula's ZFS pool.
`longhorn-garage` is a one-replica, strictly local Longhorn class for the obsync
Garage, which already replicates its data three ways itself.

Replicated storage is not a backup. It protects against losing a disk or a node, not
against deleting the data or corrupting it. Backups go to a separate target, and a
backup only counts once you have restored from it.
