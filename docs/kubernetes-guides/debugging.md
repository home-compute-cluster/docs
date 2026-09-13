# A Debugging Playbook

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

Where to look first when a Pod, a Service or the control plane misbehaves.

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

## Network debugging

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

## Control plane health

```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl top nodes
kubectl top pods -A
# from a server, check etcd peers are reachable
nc -vz <other-server> 2380
```
