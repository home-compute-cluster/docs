# Networking in Kubernetes

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

Pod addresses, Services, DNS, getting traffic in, restricting it, and how K3s and this cluster fit together.

This is the part that confuses most people, because several layers each solve a
different problem. Working from the inside out:

```
Pod IPs (CNI)  ->  Services (virtual IPs, kube-proxy)  ->  DNS (CoreDNS)
               ->  getting traffic in (NodePort, LoadBalancer, Ingress, Gateway)
               ->  restricting traffic (NetworkPolicy)
```

## The Kubernetes network model

The rules every conformant cluster follows:

- **Every Pod gets its own cluster-wide IP address.** Containers inside one Pod share
  that address and can reach each other on `localhost`.
- **All Pods can reach all other Pods**, on the same node or different nodes,
  **without NAT or proxies.**
- **Agents on a node** (the kubelet, system daemons) can reach every Pod on that node.

Kubernetes defines these rules but does not implement them itself. That is the job of
the network plugin.

## Pod networking: CNI and flannel

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

## Services: stable addresses for moving Pods

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

## DNS: finding Services by name

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

## Getting traffic into the cluster

There are several layers of increasing capability.

**NodePort.** Every node answers on a high port. Fine for testing, awkward for
anything real.

**LoadBalancer.** In a cloud, this creates a cloud load balancer. On bare metal,
something in the cluster has to provide the address. The common options:

- **ServiceLB** (bundled with K3s): uses each node's own IP and host ports. No extra
  addresses needed. Covered in detail further down this page.
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

## ServiceLB in detail (K3s-specific)

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

## NetworkPolicy: restricting traffic

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

## Ports K3s needs between nodes

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

## How this cluster puts it together

- **Public traffic** (`*.packetcraft.dev`): Cloudflare's edge terminates TLS and sends
  requests down a Cloudflare Tunnel to `cloudflared`, which runs as a Deployment inside
  the cluster and forwards to the Traefik Service at
  `http://traefik.kube-system.svc.cluster.local:80`. Traefik routes by host name with
  IngressRoutes.
- **Private traffic** (`*.lab.packetcraft.dev`): DNS points at private LAN addresses.
  Off the LAN, those are reached over Tailscale through a subnet router. Traefik is a
  LoadBalancer Service served by ServiceLB, so it answers on ports 80 and 443 of every
  node, and kube-vip also holds a shared lab address.
- **The API server** is reached through the kube-vip address `192.168.1.241`, which
  moves between the three servers.

---

## Sources

- [Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Requirements (inbound ports)](https://docs.k3s.io/installation/requirements)
- [Basic Network Options (flannel)](https://docs.k3s.io/networking/basic-network-options)
- [Networking Services (CoreDNS, Traefik, ServiceLB, network policy)](https://docs.k3s.io/networking/networking-services)
- [k3s server CLI reference](https://docs.k3s.io/cli/server)
- [kube-vip on K3s](https://kube-vip.io/docs/usage/k3s/)
