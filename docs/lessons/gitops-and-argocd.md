# Lessons: GitOps and ArgoCD

ArgoCD reconciles the cluster to whatever is in `homelab-cicd-config`. That sounds
simple, and most of the time it is. The lessons below are the cases where the loop
did not behave the way I assumed.

---

## 1. Correct in Git is not the same as correct in the cluster

**What happened.** In Week 10 the kube-prometheus-stack CRDs kept failing with
`metadata.annotations: Too long`. The fix, `ServerSideApply=true`, was already
sitting in `argocd/kube-prometheus-stack.yaml` in Git. It had simply never been
applied. In Week 11 the CNPG backup chase went through five separate failures, and
every one of them was a value that looked right in Git and was wrong or missing in
the cluster.

**The lesson.** When something is broken, check the live object first
(`kubectl get <kind> <name> -o yaml`), not the file in the repo. The file tells you
what you intended. The live object tells you what is running.

**What I do now.** Compare live against Git before changing anything, and treat
"I already committed that" as a hypothesis to verify rather than a fact.

## 2. Nothing watches `argocd/` unless you make something watch it

**What happened.** The Application manifests under `argocd/` point ArgoCD at
`apps/*`. Nothing points at `argocd/` itself. So editing an Application file and
pushing it does nothing until someone runs `kubectl apply` by hand. This came up in
Week 10 and again in Week 15 while deploying Longhorn monitoring.

**The lesson.** GitOps only covers what the root of the tree covers. If the root is
applied by hand, every change to it is a manual step that is easy to forget.

**What I do now.** Re-apply the whole Application file after any change under
`argocd/`. The real fix is an app-of-apps: one Application that watches `argocd/`,
applied once, after which everything underneath is reconciled. It is on the action
list from the September control-plane postmortem.

## 3. Synced and Healthy means the manifest is valid, not that it works

**What happened.** The kube-vip change that started the
[September control-plane outage](../postmortems/2026-09-08-deus-control-plane-outage.md)
shipped with an unsubstituted placeholder as its image tag,
`ghcr.io/kube-vip/kube-vip:<match-static-pod>`. ArgoCD synced it and reported it
Healthy, because the schema was valid. The pod crashlooped on `InvalidImageName`.
That was the third time this class of problem appeared in this cluster.

**The lesson.** ArgoCD checks that objects apply and that their resources report
ready according to its health rules. It does not know what you meant.

**What I do now.** After a sync, look at the pods, not just the Application tile.
Placeholders in templates get a grep before commit (`grep -rn '<' apps/<svc>/`).

## 4. Self-heal will fight any credential a Helm chart generates

**What happened.** The Grafana admin password reset itself on every sync for weeks.
The Grafana subchart generates a random password at `helm template` time, and
`selfHeal: true` then writes that fresh value over the live Secret on each
reconcile.

**The lesson.** If a chart owns a credential, ArgoCD will keep regenerating it.
This is not a Grafana bug, it is how templated secrets and self-heal interact.

**What I do now.** Move ownership out of the chart. A SealedSecret holds the
credential and the chart points at it with `existingSecret` (for Grafana,
`grafana.admin.existingSecret: grafana-admin`). Also note that Grafana only reads
the admin password when its database is first created, so changing it later needs
`grafana-cli admin reset-admin-password`.

## 5. Deleting an Application does not always delete what it deployed

**What happened.** In Week 8 I found the old `nginx` and `guestbook` test workloads
still running long after I thought I had removed them. Their Applications were
gone, but the Deployments, Services and namespaces were left behind as orphans.

**The lesson.** ArgoCD only cascades a delete to the managed resources when the
Application carries the `resources-finalizer.argocd.argoproj.io` finalizer, or when
you delete with cascade through the ArgoCD CLI or UI. A plain `kubectl delete` on an
Application without the finalizer just removes the Application object.

**What I do now.** Add the finalizer to Applications that should clean up after
themselves, and after removing an app, check that its namespace is really gone.

## 6. Adopt existing releases, and match the names exactly

**What happened.** kube-prometheus-stack and Longhorn were installed by hand with
Helm before ArgoCD managed them. The live kube-prometheus-stack release was called
`kube-prom`, not `kube-prometheus-stack`. Adopting it under the wrong name would
have created a second, conflicting release.

**The lesson.** ArgoCD can take over a hand-installed release without reinstalling
it, but only if chart, version, namespace and release name all match the live
release.

**What I do now.** Run `helm list -A` first, pin `targetRevision` to the version
that is actually installed, sync, confirm nothing restarts, and only then upgrade in
a separate commit. Longhorn was adopted this way in Week 10 with zero pod restarts.

## 7. Big CRD charts need server-side apply

**What happened.** Client-side apply stores the whole object in the
`kubectl.kubernetes.io/last-applied-configuration` annotation. The
kube-prometheus-stack CRDs are larger than the 262144 byte annotation limit, so the
sync failed. Switching to server-side apply then surfaced a field that was invalid
all along (`readOnlyRootFilesystem` in a pod-level `securityContext`), which
client-side apply had silently accepted.

**The lesson.** Server-side apply is not just a workaround for size. It enforces the
schema strictly, which is a good thing even when it breaks a sync that used to
"work".

**What I do now.** `ServerSideApply=true` on every CRD-heavy chart (monitoring,
Longhorn, cert-manager, CNPG), and when enabling it, strip the stale
`last-applied-configuration` annotation from the affected CRDs.

## 8. Poll-based sync turns a five minute fix into an afternoon

**What happened.** Without the GitHub webhook, ArgoCD only notices a commit on its
poll interval, which is roughly three minutes. During the Week 11 backup chase, each
fix meant push, wait, check, and often apply by hand anyway because I could not
tell whether ArgoCD had picked it up yet.

**The lesson.** Slow feedback does not just waste the waiting time. It makes you
unsure what state you are looking at, and that uncertainty is what really costs
hours.

**What I do now.** The webhook moved from "nice to have" to a real action item. In
the meantime, `argocd app sync <app>` or a refresh in the UI removes the guesswork.

## 9. Bringing a running component under GitOps is its own piece of work

**What happened.** Traefik came bundled with K3s and was not tracked by ArgoCD. In
Week 13, building the Traefik dashboard meant migrating Traefik's configuration into
the repo at the same time, and the two jobs tangled together.

**The lesson.** "Add a dashboard" and "change who manages the ingress controller"
are separate changes. Doing them together made it hard to tell which one caused
each problem.

**What I do now.** Adopt first, confirm nothing changed, then build on top. The
result was worth it: a template where a dashboard JSON file goes into a ConfigMap
and appears in Grafana, which was then reused for Longhorn, CNPG, cert-manager and
ArgoCD.
