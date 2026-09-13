# GitOps CI/CD Pipeline: Setup Guide

A reusable template for wiring any custom-built service into the homelab's
`git push -> build -> deploy` pipeline. Built around GitHub Actions (CI) +
ArgoCD/Kustomize (CD) on K3s, with `cloudflared` + Traefik for ingress.

The nginx test workload is used throughout as the worked example. To stand up a
new service, copy the structure and substitute the placeholders marked
`<like-this>`.

---

## 1. Mental model (read once)

The pipeline has two halves that never talk to each other directly. They
communicate only through the **config repo**.

```
  ┌──────────────┐  push code   ┌─────────────────┐
  │  app repo    │ ───────────> │ GitHub Actions  │   CI
  │ (source+CI)  │              │ build + push    │   (GitHub-hosted runner)
  └──────────────┘              └────────┬────────┘
                                         │ 1. push image:SHA
                                         v
                                   ┌───────────┐
                                   │  ghcr.io  │
                                   └───────────┘
                                         │ 2. write SHA into manifest
                                         v
  ┌──────────────┐ ArgoCD watches ┌─────────────┐
  │ config repo  │ <───────────── │  kustomize  │
  │ (manifests)  │                │  edit image │
  └──────┬───────┘                └─────────────┘
         │ 3. reconcile
         v
  ┌──────────────────────────────┐
  │ ArgoCD (in-cluster, always-on)│ ──> Deployment rolls to image:SHA   CD
  └──────────────────────────────┘
```

- **CI** (GitHub runner): turns a commit into a tested, versioned image and
  publishes it. Then writes the new image tag into the config repo. Has **no
  cluster access**.
- **CD** (ArgoCD, inside the cluster): watches the config repo and reconciles
  the cluster to match it. **Pulls** desired state; nothing pushes into the
  cluster from outside.

Key rule: **CD applies to everything you run; CI only applies to things you
build.** Off-the-shelf images (Grafana, Headlamp) skip CI entirely, they need
only a config-repo manifest + an ArgoCD Application.

---

## 2. Two-repo layout

Always use two repos. One repo causes an infinite loop (CI's write-back
re-triggers CI).

| Repo | Role | Who writes | Who reads |
|------|------|-----------|-----------|
| `<service>-app` | Source + CI workflow | You (humans) | GitHub Actions |
| `homelab-cicd-config` | K8s manifests | GitHub Actions (bot) | ArgoCD |

The config repo is shared across all services, each service gets its own
subfolder under `apps/`.

### App repo layout
```
<service>-app/
├── Dockerfile
├── <app source files>
└── .github/
    └── workflows/
        └── build.yaml
```

### Config repo layout (grows per service)
```
homelab-cicd-config/
└── apps/
    ├── nginx/
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   └── service.yaml
    └── <service>/
        ├── kustomization.yaml
        ├── deployment.yaml
        └── service.yaml
```

---

## 3. Config repo: the three manifest files

Per service, under `apps/<service>/`.

### `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <service>
  namespace: <service>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <service>
  template:
    metadata:
      labels:
        app: <service>
    spec:
      # No nodeSelector: let K3s place the Pod. Only pin a workload that
      # really needs one node (a GPU, or node-local storage).
      containers:
        - name: <service>
          image: ghcr.io/<owner-lowercase>/<service>-app:latest  # tag rewritten by CI
          ports:
            - containerPort: <port>
```

### `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: <service>
  namespace: <service>
spec:
  selector:
    app: <service>
  ports:
    - port: <port>
      targetPort: <port>
```

### `kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: <service>
resources:
  - deployment.yaml
  - service.yaml
images:
  - name: ghcr.io/<owner-lowercase>/<service>-app
    newTag: latest   # CI overwrites this line via `kustomize edit set image`
```

> **The `images:` block is the seam.** CI runs `kustomize edit set image` which
> surgically rewrites `newTag` here. You never hand-edit the deployment's image
> line.

#### Gotchas that cause silent failures
- **Filenames matter.** Kustomize specifically reads the file named
  `kustomization.yaml`. Don't swap file contents.
- **ghcr.io is all-lowercase.** `<owner-lowercase>` must be lowercase even if
  your GitHub login has capitals (`BarneyLaw` becomes `barneylaw`). `kustomize edit
  set image` does a **case-sensitive string match**, a case mismatch matches
  nothing and the tag silently never updates.

---

## 4. App repo: the CI workflow

`.github/workflows/build.yaml`. Substitute `<service>` and confirm the config
repo name.

```yaml
name: build-and-update-config
on:
  push:
    branches: [main]
permissions:
  contents: read
  packages: write          # lets the built-in GITHUB_TOKEN push to ghcr.io
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout app repo
        uses: actions/checkout@v4

      - name: Log in to ghcr.io
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        id: build
        run: |
          # force lowercase owner: ghcr.io rejects uppercase repo names
          IMAGE=ghcr.io/$(echo ${{ github.repository_owner }} | tr '[:upper:]' '[:lower:]')/<service>-app
          SHA=${GITHUB_SHA::7}
          docker build -t $IMAGE:$SHA -t $IMAGE:latest .
          docker push $IMAGE:$SHA
          docker push $IMAGE:latest
          echo "image=$IMAGE" >> "$GITHUB_OUTPUT"
          echo "sha=$SHA" >> "$GITHUB_OUTPUT"

      - name: Checkout config repo
        uses: actions/checkout@v4
        with:
          repository: ${{ github.repository_owner }}/homelab-cicd-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}
          path: config

      - name: Install kustomize
        run: |
          curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
          sudo mv kustomize /usr/local/bin/

      - name: Bump image tag
        working-directory: config/apps/<service>
        run: |
          kustomize edit set image ${{ steps.build.outputs.image }}=${{ steps.build.outputs.image }}:${{ steps.build.outputs.sha }}

      - name: Commit and push tag bump
        working-directory: config
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add apps/<service>/kustomization.yaml
          git commit -m "deploy: <service> ${{ steps.build.outputs.sha }}" || echo "no changes"
          git push
```

Image is tagged by **short SHA** (immutable, traceable) and `latest`
(convenience). ArgoCD deploys the SHA, so every deploy is pinned to an exact
build.

---

## 5. Credentials (the part that's easy to get wrong)

Three boundaries, each needs its own credential **only when it crosses into
private**. Scope every credential to the minimum.

| Boundary | Credential | Who creates it | Needed when |
|----------|-----------|----------------|-------------|
| CI to ghcr.io (push image) | `GITHUB_TOKEN` (auto) + `packages: write` | GitHub, automatic | always (just declare the permission) |
| CI to config repo (write tag) | `CONFIG_REPO_TOKEN` (fine-grained PAT) | You, by hand | always (cross-repo write) |
| Cluster to ghcr.io (pull image) | image pull secret | You | only if package is **private** |
| ArgoCD to config repo (read) | repo creds | You | only if config repo is **private** |

### Creating `CONFIG_REPO_TOKEN` (one-time, reusable across all services)
1. GitHub, then your avatar, then **Settings**, then **Developer settings**.
2. **Personal access tokens**, then **Fine-grained tokens**, then **Generate new token**.
3. **Repository access:** Only select repositories, then pick `homelab-cicd-config`.
4. **Permissions:** under Repository permissions, set **Contents** to **Read and write**.
   Nothing else.
5. Generate, copy the `github_pat_...` value.
6. In **each** `<service>-app` repo, go to Settings, then Secrets and variables,
   then Actions, then New repository secret. Name: `CONFIG_REPO_TOKEN`, value: the token.

> One PAT scoped to the config repo works for every app repo, since they all
> write to the same config repo. Add it as a secret in each app repo.

### Private package: image pull secret
If the ghcr package is private (default), the cluster can't pull. Either make it
public (the package settings, then Change visibility, then Public), or:
```bash
kubectl create secret docker-registry ghcr-pull \
  --namespace <service> \
  --docker-server=ghcr.io \
  --docker-username=<owner> \
  --docker-password=<PAT-with-read:packages>
```
then add to the deployment pod spec:
```yaml
    spec:
      imagePullSecrets:
        - name: ghcr-pull
```

### Private config repo: ArgoCD repo creds
```bash
argocd repo add https://github.com/<owner>/homelab-cicd-config.git \
  --username <owner> \
  --password <PAT-with-repo-read>
```

---

## 6. CD: the ArgoCD Application

Per service. Save as `<service>-app.yaml`, apply with `kubectl apply -f`. For
full GitOps, also commit this into the config repo (e.g. `argocd/<service>.yaml`)
so ArgoCD's own app definitions are reproducible.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <service>
  namespace: argocd          # the Application object lives in argocd's namespace
spec:
  project: default
  source:
    repoURL: https://github.com/<owner>/homelab-cicd-config.git
    targetRevision: main
    path: apps/<service>     # the folder ArgoCD watches
  destination:
    server: https://kubernetes.default.svc   # "this cluster"
    namespace: <service>     # where the workload's pods land
  syncPolicy:
    automated:
      prune: true            # delete resources removed from Git
      selfHeal: true         # revert manual drift back to Git's state
    syncOptions:
      - CreateNamespace=true # create the namespace if absent
```

```bash
kubectl apply -f <service>-app.yaml
```

> **`repoURL` is the entire Gitea migration later**, switch this one line to
> the Gitea URL, re-add repo creds, done.

---

## 7. Ingress (optional, for externally-reachable services)

Architecture: the Cloudflare edge sends requests down the tunnel to the cluster's
`cloudflared` Deployment, which forwards them to the Traefik Service, where an
IngressRoute sends them on to your Service. `cloudflared` runs inside K3s (two
replicas, schedulable on any node), so the tunnel does not depend on a single node.
The tunnel is a dumb pipe; Traefik is the routing brain.

1. Add a public hostname to the cluster's tunnel in Cloudflare:
   `<service>.packetcraft.dev` pointing at the Traefik Service, for example
   `http://traefik.kube-system.svc.cluster.local:80`. Cloudflare auto-creates
   a proxied CNAME and terminates TLS at its edge (real cert, required because
   `.dev` is HSTS-preloaded and rejects self-signed).
2. Create a Traefik IngressRoute:
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: <service>
  namespace: <service>
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`<service>.packetcraft.dev`)
      kind: Rule
      services:
        - name: <service>
          port: <port>
```
Every new externally-exposed service = one CNAME (public hostname on the tunnel)
+ one IngressRoute. The tunnel config itself never changes again.

> If the app serves its own TLS / redirects to HTTPS internally (e.g. ArgoCD),
> run it in insecure/HTTP mode so Traefik+Cloudflare own TLS, or you get a
> redirect loop.

---

## 8. End-to-end test (proves the loop)

```bash
# 1. make a visible change in the app repo, push to main
#    (edit index.html / app source), git commit, git push

# 2. GitHub Actions tab: build-and-update-config runs green
#    - builds image:SHA, pushes to ghcr
#    - commits "deploy: <service> <sha>" to homelab-cicd-config

# 3. config repo: confirm the bot commit bumped newTag

# 4. ArgoCD detects change (~3 min poll; force with `argocd app sync <service>`)
kubectl get application <service> -n argocd      # Synced / Healthy
kubectl get pods -n <service> -o wide            # new pod, on whichever node K3s chose
kubectl rollout status deployment/<service> -n <service>
```

### Prove the GitOps guarantee (self-heal)
```bash
kubectl scale deployment/<service> -n <service> --replicas=3
# within ~1 min ArgoCD reverts to replicas=1 (the git-declared value)
kubectl get deployment <service> -n <service>
```

---

## 9. New-service checklist

For each new built-from-source service:

- [ ] Create `<service>-app` repo with `Dockerfile`, source, `.github/workflows/build.yaml`.
- [ ] Add `CONFIG_REPO_TOKEN` secret to the app repo.
- [ ] Substitute `<service>`, `<port>`, `<owner-lowercase>` in workflow + manifests.
- [ ] Add `apps/<service>/` (deployment, service, kustomization) to `homelab-cicd-config`.
- [ ] Push the app repo, confirm Actions is green, and confirm the bot commit in the config repo.
- [ ] Make ghcr package public (or add pull secret).
- [ ] Apply the ArgoCD Application and confirm Synced/Healthy, pod Running.
- [ ] (Optional) Add tunnel hostname + IngressRoute for external access.
- [ ] Run the end-to-end test + self-heal proof.

For off-the-shelf images (no source build): skip the app repo and CI entirely.
Just add `apps/<service>/` to the config repo pointing at the public image, and
apply the ArgoCD Application.

---

## 10. Common failure modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `repository name must be lowercase` | uppercase owner in image | force lowercase with `tr` in workflow |
| Build green but no deploy | ArgoCD Application not created, or `images:` name mismatch (case) | create Application; align `name:` exactly, lowercase |
| Checkout config repo fails | wrong repo name or missing/expired PAT | fix `repository:`; regenerate `CONFIG_REPO_TOKEN` |
| `ImagePullBackOff` / `ErrImagePull` | package is private | make public or add image pull secret |
| ArgoCD `Unknown` / repo error | private config repo, no creds | `argocd repo add` with read PAT |
| Browser HSTS / self-signed error on `.dev` | not going through the tunnel (stale hosts entry, DNS-only record) | remove hosts entry; ensure proxied CNAME; verify edge cert with `curl -sv ... | grep issuer` |
| Redirect loop behind tunnel | app serves its own HTTPS | run app in insecure/HTTP mode |
| Tag never updates | `kustomize edit set image` name mismatch | `name:` in kustomization must exactly match built image (case-sensitive) |
