# Deployment Modes

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

The ways to run Kubernetes, and the cluster shapes K3s supports, from a single SQLite server to HA embedded etcd.

## Ways to run Kubernetes in general

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

## K3s cluster shapes

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

## Choosing

| Situation | Shape |
|---|---|
| Learning, CI, a single box | single server, SQLite |
| Small cluster that must survive losing a server | three servers, embedded etcd |
| You already run a reliable SQL database | two or more servers, external datastore |
| Control plane on weak hardware or doing critical work | tainted dedicated servers plus agents |

---

## Sources

- [Architecture](https://docs.k3s.io/architecture)
- [Cluster Datastore](https://docs.k3s.io/datastore)
- [High Availability Embedded etcd](https://docs.k3s.io/datastore/ha-embedded)
