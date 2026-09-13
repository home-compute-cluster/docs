# Lessons Learnt

Four months of building and breaking this cluster produced a lot of hard-won
knowledge, but it was scattered across weekly diaries, guides and postmortems. This
section pulls it together by topic so it can actually be reused.

Each lesson follows the same pattern: what happened, what it taught me, and what I
do differently now. Where a lesson came out of a specific week or incident, it says
so, so you can go back and read the full story.

## The topics

| Page | What it covers |
|------|----------------|
| [GitOps and ArgoCD](gitops-and-argocd.md) | The gap between what Git says and what the cluster runs, self-heal, CRD-heavy charts, adoption |
| [Secrets and credentials](secrets-and-credentials.md) | Sealed Secrets, naming, S3 keys, and credentials that leak in through side doors |
| [Networking and remote access](networking-and-remote-access.md) | Tailscale subnet routing, return paths, TLS and DNS, verifying network changes |
| [Storage and backups](storage-and-backups.md) | Longhorn, ZFS, used disks, and why a backup is not proven until it is restored |
| [Control plane and nodes](control-plane-and-nodes.md) | etcd quorum, SQLite surprises, monitoring blind spots, configuration drift |
| [Debugging and incident response](debugging-and-incident-response.md) | Reading logs that say nothing, hardware hangs, and keeping discipline under pressure |
| [Planning and working habits](planning-and-working-habits.md) | Sequencing, automation, AI tools, and doing the boring things first |

## If you only read five

1. **Correct in Git is not the same as correct in the cluster.** Most of the long
   debugging sessions came from a change that was committed but never applied.
2. **A fault that survives a reboot is written down somewhere on disk.** Stop
   looking at hardware and start reading the configuration that loads at boot.
3. **Anything configured by hand has an unknown value.** Two of the three
   postmortems trace back to settings that were never in Ansible.
4. **A backup you have never restored is a hope, not a backup.**
5. **Your monitoring should not live behind the thing it monitors.** When the
   tailnet path died, it took Grafana, Headlamp and Alertmanager with it.
