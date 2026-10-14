# Lessons: Networking and Remote Access

Remote access to the cluster goes through Tailscale: deus advertises
`192.168.1.0/24` as a subnet router, and the `*.lab.packetcraft.dev` names resolve
to private addresses that only work over the tailnet. Two of the three postmortems
were caused by this setup, so this page is mostly about getting it right.

---

## 1. A subnet router must never accept routes for the prefix it advertises

**What happened.** deus was advertising `192.168.1.0/24` and also had
`--accept-routes` enabled. When kube-vip changed the addresses on `eno1`,
`tailscaled` reconfigured and installed that same prefix into its routing table
(table 52) pointing at `tailscale0`. From then on deus could send traffic but could
not answer it: replies to LAN hosts went into the tunnel instead of back out the
network card. SSH, ping, the Kubernetes API and etcd all failed together for over
three days. See the
[control-plane outage postmortem](../postmortems/2026-09-08-deus-control-plane-outage.md).

A week earlier, sol and opus had the same problem in a milder form. They accepted
deus's advertisement and started routing their own LAN traffic, including etcd peer
traffic, through the tunnel to deus and back. Nodes flapped between Ready and
NotReady. See section 9 of the
[deus lockup postmortem](../postmortems/2026-09-02-deus-lockup.md).

**The lesson.** On this network, cluster nodes need tailnet identities, not accepted
routes. The subnet router advertises without accepting, and that keeps all the
functionality.

**What I do now.**

```bash
sudo tailscale set --accept-routes=false
ip route show table 52        # must not contain 192.168.1.0/24
```

Use `tailscale set`, not `tailscale up`. `up` resets every flag you do not repeat,
which is one likely way `--accept-routes` got switched on without anyone choosing
it. Also note that `tailscale status --json | jq '.Self.RouteAll'` always returns
`null`, so it cannot be used to check this setting.

## 2. One subnet router means one node can take away all remote access

**What happened.** When deus hung on 2 September, every other node kept serving
public traffic through the Cloudflare tunnel, but none of them could be reached for
administration. SSH, `kubectl`, ArgoCD, Grafana and Headlamp all depended on that one
advertisement.

**The lesson.** Redundancy in the workloads does not help if the only way in has
none.

**What I changed.** sol now advertises `192.168.1.0/24` as well, so deus is one of
two subnet routers. Tailscale
elects one primary for a prefix and fails over to another advertiser if it goes
offline. Before advertising on a new node, IP forwarding must be enabled, or the
elected router will silently drop the whole prefix:

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
sudo tailscale set --advertise-routes=192.168.1.0/24
```

## 3. Diagnostic tools must not sit behind the thing they diagnose

**What happened.** Grafana, Headlamp and Alertmanager are all lab plane services. The
outage that removed the lab plane also removed every tool I would have used to see
it, and Alertmanager could not have sent a notification even if a rule had fired.

**What I do now.** Treat "how would I find out if the tailnet path died" as a design
question. Alertmanager now pushes to a Discord channel, which goes out over the
internet and does not need the tailnet, so a firing alert reaches me even when the
lab plane is gone. What it cannot do yet is report its own death: the always-firing
`Watchdog` alert is still routed to nowhere, and an external heartbeat check is what
would close that gap.

## 4. "Can ping out but nobody can ping in" is a return path problem

**What happened.** From the console, deus could reach the internet and other LAN
nodes. Nothing could reach deus. `tcpdump` showed incoming pings arriving, but no
replies went back.

**The lesson.** `tcpdump` captures below routing and filtering, so seeing the packet
arrive rules out cables, switch ports and the NIC in one step. What is left is a
decision the kernel makes about a packet it already has, which points straight at
`ip route` and `ip rule`.

**What I do now.** When reachability is one-directional, go to the routing tables
first:

```bash
ip rule
ip route get <source-ip>      # where would the reply go?
ip route show table all | grep <prefix>
```

## 5. Changing one interface can wake up every daemon that watches interfaces

**What happened.** kube-vip did nothing wrong on its own. It added an address to
`eno1`. That address change made `tailscaled` run a full reconfigure, and the latent
`--accept-routes` problem became live. A DHCP renewal, a NIC reset or an Ansible run
touching networking could have done the same.

**The lesson.** On a node running Tailscale, kube-vip, flannel and a firewall, a
network change is never local to one tool.

## 6. Verify a network change on a server node before walking away

**What happened.** The kube-vip change was pushed and not checked. The failure began
twelve minutes later.

**What I do now.** After any networking, Tailscale or firewall change on a server
node, confirm etcd can still talk to its peers:

```bash
nc -vz 192.168.1.251 2380
nc -vz 192.168.1.252 2380
kubectl get nodes
```

## 7. Firewall state is configuration too

**What happened.** During the same outage, `ufw` turned out to be active on deus,
even though the intended baseline for cluster nodes is disabled. It was blocking
TCP 2380 from both etcd peers. Nobody knows when it was switched on.

**What I am changing.** The firewall state is being added to the Ansible base role
so it is asserted on every run instead of assumed. K3s's own documentation lists the ports a server needs:
2379-2380 between servers for etcd, 6443 for the API, 8472/udp for flannel VXLAN and
10250 for the kubelet.

## 8. `.dev` domains need real certificates everywhere

**What happened.** `.dev` is on the HSTS preload list, so browsers refuse plain HTTP
and self-signed certificates with no way to click through. This forced the
Cloudflare tunnel for public names in Week 5, and a real Let's Encrypt wildcard for
`*.lab.packetcraft.dev` through cert-manager DNS-01 in Week 8.

**The lesson.** Choosing a TLD is an infrastructure decision.

## 9. MagicDNS cannot hold your own records

**What happened.** I assumed the `lab.` names could live in Tailscale's DNS. MagicDNS
only resolves tailnet device names, not arbitrary custom records.

**What I do now.** Cloudflare holds DNS-only (grey cloud) A records for the lab
names. They resolve publicly, but the private addresses they point at only work from
inside the tailnet.

## 10. Every address you use to reach the API must be in the certificate

**What happened.** Pointing a kubeconfig at a node's LAN IP failed until that IP was
added to `--tls-san` on the server. The same applies to the kube-vip address
`192.168.1.241`.

**What I do now.** Any new way of reaching the API (a VIP, a tailnet IP, a DNS name)
gets added to `tls-san` in the K3s config first, then used.

## 11. `localhost` over SSH is the remote machine's localhost

**What happened.** In Week 4, `kubectl port-forward` on deus followed by opening
`localhost:8080` on my laptop did nothing, because the port was bound on deus.

**What I do now.** Run `kubectl` from my own machine with a copied kubeconfig, so
`localhost` means what I expect. Over the tailnet the same kubeconfig works at home
and away.
