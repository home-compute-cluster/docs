# Packaging and Common Patterns

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

Helm, Kustomize and K3s's Helm integration, the workload shapes that come up again and again, and a small end-to-end example.

## Helm, Kustomize and K3s's Helm integration

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

## Common workload shapes

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

## Minimal end-to-end example

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

## Sources

- [Managing Packaged Components](https://docs.k3s.io/installation/packaged-components)
