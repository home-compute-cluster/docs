# Lessons: Planning and Working Habits

Not every lesson is about a YAML field. Some of the most useful ones are about the
order to do things in, and the habits that keep a side project alive alongside
school, interviews and everything else.

---

## 1. Do not stack several new problems in one step

**What happened.** obsync needs Go, Postgres and a hand-written sync protocol. The
portfolio site needs Go and a few JSON endpoints. Doing the portfolio site first meant
learning Go and the deployment shape with low stakes, before adding distributed
systems correctness on top.

**What I do now.** When a project brings more than one unfamiliar thing, find a
smaller project that brings just one of them and do that first. obsync followed the
same idea: a simple pull-based Canvas mirror first, real-time sync later.

## 2. Never let AI fully design your application architecture

**What happened.** The portfolio site backend took far longer than it should have in
Week 14 because its architecture was odd and hard to work with. A lot of that design
came from accepting AI suggestions without thinking them through.

**The lesson.** AI tools are good at filling in pieces. They are not accountable for
how the pieces fit together, and you are the one who has to maintain it.

**What I do now.** Settle the structure (layers, data flow, what talks to what)
myself before asking for help with the parts. Finish the frontend's shape before
committing to a backend design.

## 3. Automate before the node count makes manual work painful

**What happened.** The first three nodes were set up by hand over SSH. By the time
sol and opus arrived, I decided they had to be provisioned with Ansible. It took
longer than expected because of the SQLite to etcd migration, but the result is that
a new node now only needs an OS, SSH access and a static IP in netplan.

**The lesson.** Manual setup is fine for learning what the steps are. After that,
every manual step is toil and every hand-configured node is a slightly different
node.

## 4. Write the guide while the pain is fresh

**What happened.** The CI/CD guide was written in the same week the pipeline first
worked. Every service after that was a checklist rather than a rediscovery.

**What I do now.** When something takes more than an evening to get working, it gets
a guide or a lesson entry before moving on. Guides also need updating as the setup
changes. The deployment guide still does not cover secrets and IngressRoutes, which
was the main blocker when deploying the portfolio backend.

## 5. Get the physical layer right before touching software

**What happened.** Week 1 was entirely coax splitters, MoCA adapters and ping tests.
A flaky link would have shown up later as a hundred confusing software problems. In
Week 11 the cluster moved to a direct connection to the router, removing the MoCA hop
entirely.

**The lesson.** Fewer boxes in the path means fewer things to suspect.

## 6. Never expose an admin UI before it has authentication

**What happened.** In Week 7 I put the Longhorn UI behind a public tunnel hostname
without noticing it has no login. It was taken down quickly, and from then on Longhorn
was only reached through port-forwarding.

**What I do now.** Before any hostname goes public, check whether the app has its own
auth. Anything with write access to the cluster stays on the tailnet plane only.

## 7. Write down what you are deliberately leaving open

**What happened.** Every weekly diary ends with a "still open" list. That habit is
why the webhook, the app-of-apps and the restore drill are still visible as gaps
instead of forgotten.

**The lesson.** An open item with a reason attached is a decision. An open item with
no record is a surprise waiting to happen.

## 8. The cluster has to survive the weeks you are not looking at it

**What happened.** Weeks 12 to 14 were taken up by interviews, a new semester and
software projects. Daily manual status checks were the only monitoring. Alerting was
"as soon as possible" for weeks, and in the end it was the September incidents that
showed how much it was needed.

**The lesson.** A homelab you only understand while actively working on it is not
really operated. Alerting and automated patching are what let it run while life
happens.

## 9. Change the plan when the facts change, and say so

**What happened.** Several plans got dropped along the way: MetalLB was never needed
because ServiceLB already did the job, Jellyfin was dropped as a goal, and the idea
that nebula "owns data" while K3s "owns services" turned out not to describe the
cluster at all.

**The lesson.** Keeping a plan written down is only useful if you are willing to
cross things out. Documentation that describes the plan instead of the system is
worse than none, because people trust it.
