# kubectl: the Commands You Will Actually Run

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

The everyday command line, plus a quick reference card at the end.

The pattern is almost always `kubectl <verb> <resource> [name] [flags]`.

## Inspection

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

## Interacting with running Pods

```bash
kubectl exec -it <pod> -n <ns> -- sh
kubectl exec <pod> -n <ns> -- nginx -t
kubectl cp <pod>:/etc/nginx/nginx.conf ./out.conf -n <ns>
kubectl debug -it <pod> -n <ns> --image=busybox:1.36   # temporary debug container
```

## Port-forwarding, and the SSH trap

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

## Applying and deleting

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

## Rollouts and scaling

```bash
kubectl rollout status deployment/<svc> -n <ns>
kubectl rollout history deployment/<svc> -n <ns>
kubectl rollout undo deployment/<svc> -n <ns>
kubectl rollout restart deployment/<svc> -n <ns>
kubectl scale deployment/<svc> -n <ns> --replicas=3
```

## Quality of life

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
