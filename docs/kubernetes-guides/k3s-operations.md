# K3s Operations and Observability

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

Configuring, backing up, upgrading and watching a running K3s cluster.

## K3s operations

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

## Observability basics

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

## Sources

- [Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Managing Packaged Components](https://docs.k3s.io/installation/packaged-components)
- [k3s server CLI reference](https://docs.k3s.io/cli/server)
- [etcd-snapshot](https://docs.k3s.io/cli/etcd-snapshot)
- [Automated Upgrades](https://docs.k3s.io/upgrades/automated)
