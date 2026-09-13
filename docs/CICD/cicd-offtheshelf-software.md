# GitOps for Off-the-Shelf Software: Setup Guide

A companion to `cicd-pipeline-setup-guide.md`. That guide covers
**built-from-source** services (you write the code, CI builds an image). This one
covers **off-the-shelf software** you don't compile, Grafana, Headlamp,
Longhorn, cert-manager, anything that already ships an image or a Helm chart.

The key difference: **off-the-shelf software has no app repo and no CI.** There is
nothing to build. You only declare *what to run*; ArgoCD reconciles the cluster to
match. This is the `CD applies to everything you run; CI only applies to things
you build` rule from the main guide, here, CI is simply absent.

---

## 1. Which of the two patterns applies?

Off-the-shelf software arrives in one of two shapes. Pick the pattern that matches
how the software is distributed.

| Shape | Example | Pattern |
|-------|---------|---------|
| A plain container image | Uptime Kuma, a public nginx, linuxserver.io images | **Pattern A: raw manifests** |
| A Helm chart | kube-prometheus-stack, Longhorn, cert-manager, ArgoCD itself | **Pattern B: ArgoCD Application pointing at a Helm chart** |

Rule of thumb: if the project's install docs say "here's our image, write your own
Deployment," use Pattern A. If they say "`helm install ...`," use Pattern B, never
hand-translate a chart into raw manifests, you lose every upstream default and
every future upgrade.

---

## 2. Pattern A: raw manifests for a plain image

Identical to the source-built layout in the main guide, minus the app repo and the
`images:` tag-bump seam (there's no CI to bump it, you pin the version yourself).

### Config repo layout
```
homelab-cicd-config/
└── apps/
    └── uptime-kuma/
        ├── kustomization.yaml
        ├── deployment.yaml
        └── service.yaml
```

### `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  replicas: 1
  selector:
    matchLabels:
      app: uptime-kuma
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      # placement: only add a nodeSelector if the workload really belongs on one node
      containers:
        - name: uptime-kuma
          image: louislam/uptime-kuma:1.23.16   # PIN A VERSION, never :latest
          ports:
            - containerPort: 3001
```

### `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  selector:
    app: uptime-kuma
  ports:
    - port: 3001
      targetPort: 3001
```

### `kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: uptime-kuma
resources:
  - deployment.yaml
  - service.yaml
```

> No `images:` block needed, there's no CI rewriting a tag. You set the version in
> `deployment.yaml` by hand and commit it. **Renovate** (already on your roadmap)
> watches these pinned tags and opens a PR when a newer `louislam/uptime-kuma` tag ships,
> so upgrades stay declarative: review PR → merge → ArgoCD rolls it.

### ArgoCD Application
Same as any service (from the main guide §6):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: uptime-kuma
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/BarneyLaw/homelab-cicd-config.git
    targetRevision: main
    path: apps/uptime-kuma
  destination:
    server: https://kubernetes.default.svc
    namespace: uptime-kuma
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 3. Pattern B: ArgoCD Application pointing at a Helm chart

This is the important one, because most serious off-the-shelf infra ships as a
chart, and **it's how Grafana already arrived on your cluster** (via
kube-prometheus-stack). The mistake to avoid: `helm install` by hand. That puts the
release outside Git, exactly the drift that bit the Grafana `lab.` IngressRoute.

ArgoCD renders Helm charts itself. You point an Application at the chart and supply
values; ArgoCD templates it and reconciles the output. The chart's release is now
**owned by Git**, upgrades are a values/version change in a commit, and self-heal
applies.

### Two ways to feed values

**B1: inline values in the Application** (fine for small overrides):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 65.1.1            # PIN the chart version
    helm:
      values: |
        grafana:
          enabled: true
          # values that used to live in your hand-run `helm install`
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true          # big CRD-heavy charts need this
```

**B2: values file in the config repo** (better once values grow; this is the
clean long-term shape). Use a multi-source Application: one source is the upstream
chart, the other is your repo holding the values file.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 65.1.1
      helm:
        valueFiles:
          - $values/apps/kube-prometheus-stack/values.yaml
    - repoURL: https://github.com/BarneyLaw/homelab-cicd-config.git
      targetRevision: main
      ref: values                     # makes $values resolve to this repo
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```
Config repo gains:
```
homelab-cicd-config/
└── apps/
    └── kube-prometheus-stack/
        └── values.yaml      # everything you'd have passed to `helm install`
```

> **Migrating an already-hand-installed chart (your Grafana case):** point a
> Pattern B Application at the same chart + version you currently have installed,
> with your current values. On first sync ArgoCD adopts the existing release rather
> than duplicating it, *provided* chart name, version, namespace, and release name
> line up. Pin `targetRevision` to the version you actually have now (find it with
> `helm list -A`), sync, confirm Healthy, *then* bump the version in a later commit.
> Don't combine "adopt" and "upgrade" in one move, you won't know which broke what.

### Where the Grafana `lab.` IngressRoute belongs now

Two clean homes, pick one:

1. **In the chart values**, kube-prometheus-stack exposes
   `grafana.extraObjects` (or you template the IngressRoute through the chart),
   so the `lab.` IngressRoute is rendered by the same Application. Most cohesive.
2. **As a sibling raw manifest**, a small `apps/monitoring-extras/` folder
   (Pattern A, manifests only, no Deployment) holding just the Grafana
   IngressRoute, with its own ArgoCD Application. Simpler to reason about; keeps
   "the chart" and "my routing for the chart" as separate Applications.

Either way the hand-applied IngressRoute stops being an orphan. Given you already
have it applied, the lowest-friction path is option 2: drop the YAML you already
wrote into `apps/monitoring-extras/`, add an Application, and ArgoCD adopts it.

---

## 4. Decision flow (the whole thing in one place)

```
New software to run?
│
├─ Do you write its source code?
│   ├─ YES → main guide (app repo + CI + config repo). CI builds & bumps tag.
│   └─ NO  → this guide (config repo only, no CI):
│           │
│           ├─ Distributed as a Helm chart?
│           │   ├─ YES → Pattern B (ArgoCD App → chart + pinned version + values)
│           │   │         small values → B1 inline; growing values → B2 values file
│           │   └─ NO (plain image) → Pattern A (raw manifests, pinned tag)
│           │
│           └─ Keep upgrades declarative with Renovate watching pinned versions.
```

---

## 5. Off-the-shelf checklist

- [ ] Decide Pattern A (image) vs B (chart). When in doubt, follow how upstream
      tells you to install it.
- [ ] **Pin the version**: image tag (A) or `targetRevision` chart version (B).
      Never `:latest`, never an unpinned chart.
- [ ] Add `apps/<service>/` to `homelab-cicd-config` (manifests, or chart values).
- [ ] Create the ArgoCD Application (commit it into the repo too, e.g.
      `argocd/<service>.yaml`, for reproducibility).
- [ ] For charts: add `ServerSideApply=true` if it's CRD-heavy (prometheus stack,
      Longhorn, cert-manager all qualify).
- [ ] Confirm `Synced / Healthy`, pods Running.
- [ ] (Optional) tunnel hostname + IngressRoute for public, or `lab.` IngressRoute
      + grey-cloud Cloudflare A record for tailnet-only.
- [ ] Let Renovate watch the pinned version for upgrade PRs.

---

## 6. Common failure modes (additions to the main guide's table)

| Symptom | Cause | Fix |
|---------|-------|-----|
| Chart sync fails on CRDs / `the server could not find the requested resource` | giant chart applied client-side; CRDs too big for annotations | add `ServerSideApply=true` to `syncOptions` |
| ArgoCD duplicates a chart you `helm install`ed by hand | release name / namespace / version don't match the existing release | align all three to the live release (`helm list -A`) so ArgoCD adopts, not re-creates |
| Hand-applied manifest vanishes after a `helm upgrade` | the object lived outside Git, inside a chart-managed namespace | move it into the config repo as a tracked manifest or into chart values |
| `:latest` silently changed under you | unpinned image tag | pin an explicit version; let Renovate PR upgrades |
| Application stuck `OutOfSync` on a chart's own mutated fields | chart writes fields back (webhooks, defaulting) | add the noisy fields to `ignoreDifferences` on the Application |
