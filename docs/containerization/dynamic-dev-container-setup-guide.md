# Dynamic Dev Container — Setup Guide

A disposable, cluster-scheduled VS Code dev environment that lands on whichever
always-on node has capacity, deployed through the same `git push → ghcr →
ArgoCD` pipeline as everything else. Doubles as a contained sandbox for letting
an AI agent run without reaching the host or your LAN.

**Two goals, one design.** The disposability that makes this a clean
platform-engineering pattern (stateless, scheduled, reproducible) is the same
disposability that makes it a safe AI sandbox (blast radius resets to last `git
push`; the network policy keeps the agent off your home network).

---

## 1. How it works

```
   local VS Code (Remote-Tunnels extension)
        │  attaches to the named tunnel "homelab-dev"
        ▼
   Microsoft tunnel relay  ◀── code tunnel (in the pod) phones out on 443
        ▲
        │  pod scheduled by K3s onto deus OR nebula (prefers nebula)
   ┌────┴──────────────────────────────────────────┐
   │  code-server pod                               │
   │   • init container: git clone → emptyDir       │
   │   • workspace = EPHEMERAL (resets on restart)  │
   │   • NetworkPolicy: DNS + cluster + internet,   │
   │     but LAN (192.168/16) BLOCKED               │
   │   • non-root, caps dropped, resource ceilings  │
   └────────────────────────────────────────────────┘
```

You connect to the **cluster**, not a host. The scheduler places the pod; your
editor follows the tunnel, not an IP. That's the "dynamic" you asked for.

### Why `code tunnel`, not code-server (browser)
You wanted your real laptop VS Code, not a browser tab. `code tunnel` runs
Microsoft's tunnel host inside the pod; your desktop VS Code's **Remote-Tunnels**
extension attaches to it by name. The pod can be anywhere — the relay handles
reachability, so no ingress/IngressRoute is even needed for the editor itself.

---

## 2. The "dynamic placement" — what it does and doesn't do

Honest framing, because the word oversells easily:

- **Hard rule:** the pod is schedulable **only on `deus` or `nebula`**, never
  `legion` (transient — it'd evict your editor when the GPU node sleeps).
- **Soft rule:** prefer `nebula` (more RAM for compile/AI), fall back to `deus`.
- Placement is decided **once, at schedule time**, from `resources.requests` +
  the preference. K3s does **not** rebalance a running pod. So "whichever is
  less loaded" means *less loaded at the moment the pod is created*, not a live
  load balancer. Restart the pod (delete it; the Deployment recreates it) to
  re-trigger placement.

This is the genuine, defensible version of dynamic — the same model real
platform teams use. Live editor-session migration across nodes isn't a thing
that exists cleanly; "stateless pod, reschedule freely" is the honest best.

---

## 3. Repos & files (mirrors your existing two-repo pattern)

**`code-server-app`** (new app repo — source + CI):
```
code-server-app/
├── Dockerfile               # toolchain baked in; runs `code tunnel` as non-root
├── entrypoint.sh            # clone workspace, start tunnel
└── .github/workflows/build.yaml   # same CI shape as nginx-app
```

**`homelab-cicd-config`** (existing config repo — add one folder):
```
apps/code-server/
├── namespace.yaml
├── pvc.yaml                 # tiny, tunnel-auth only (see §6 for the tradeoff)
├── networkpolicy.yaml       # egress containment
├── deployment.yaml          # affinity + hardening + ephemeral workspace
└── kustomization.yaml
argocd/
└── code-server.yaml         # the ArgoCD Application
```

---

## 4. Placeholders to substitute

Before pushing, replace these across the files:

| Placeholder | Where | Value |
|-------------|-------|-------|
| `<owner>` | argocd app, deployment env, ci workflow | your GitHub username (as-is) |
| `<owner-lowercase>` | kustomization, deployment image | username **lowercased** |
| `<your-dev-repo>` | deployment.yaml (init + main `REPO_URL`) | the repo you want to hack on in the sandbox |
| node names `deus`/`nebula` | deployment.yaml affinity | confirm against `kubectl get nodes` |

> The lowercase rule is the same silent-failure trap from your nginx guide:
> `kustomize edit set image` is a case-sensitive string match. Mismatch =
> tag never updates, no error.

---

## 5. Standing it up (order matters)

```bash
# 1. Create the app repo on GitHub, push the Dockerfile + entrypoint + workflow.
#    Add CONFIG_REPO_TOKEN secret to it (reuse the same PAT as nginx-app —
#    it's already scoped to homelab-cicd-config).

# 2. Push apps/code-server/ + argocd/code-server.yaml to homelab-cicd-config.

# 3. First CI run builds the image. The dev image is BIG (Go+Rust+kubectl+
#    VS Code CLI) — expect the first build to take a few minutes.

# 4. Make the ghcr package public (simplest) OR add a pull secret:
#    GitHub → Packages → code-server-app → settings → Change visibility → Public

# 5. Register the ArgoCD Application:
kubectl apply -f argocd/code-server.yaml

# 6. Watch it land:
kubectl get application code-server -n argocd      # Synced / Healthy
kubectl get pods -n code-server -o wide            # note which node it landed on
```

### First-time tunnel login (one-time, see §6)
`code tunnel` needs to authenticate once. Read the device-login URL from the
pod logs and complete it in a browser:

```bash
kubectl logs -n code-server deploy/code-server -f
# look for:  "To grant access ... open https://github.com/login/device
#             and use code XXXX-XXXX"
```

After login, the tunnel "homelab-dev" appears in your laptop VS Code:
**Command Palette → Remote-Tunnels: Connect to Tunnel → homelab-dev**.

---

## 6. The one real wrinkle: ephemeral workspace vs. tunnel auth

The workspace is **deliberately ephemeral** (emptyDir): every pod restart
re-clones from git, resetting the AI's blast radius. Good.

But `code tunnel`'s *auth token* also lives in the container. If everything is
ephemeral, every reschedule forces a fresh GitHub device-login. So the design
persists **only the tiny auth blob** (a 64Mi PVC mounted at `~/.vscode-cli`)
while keeping the workspace ephemeral.

**The catch:** K3s `local-path` pins that PVC to one node. Once the pod creates
the volume on (say) nebula, future reschedules stick to nebula — which partly
defeats dynamic placement. Your three options, spelled out in `pvc.yaml`:

- **(A) Keep the PVC** — first placement dynamic, then sticky; seamless
  reconnect. *Default.*
- **(B) Delete the PVC + volume/mount** — fully stateless, placement dynamic on
  every restart, but re-login each cycle (one URL in the logs).
- **(C) Longhorn PVC later** (NAS-gated) — node-independent replicated storage
  gives *both* persisted auth and free placement. The correct long-term answer.

If dynamic placement matters more to you than skipping the occasional re-login,
start with **(B)** — it's the purest expression of the thesis. You can move to
(A) or (C) anytime; it's a volume swap, not a redesign.

---

## 7. What's contained, and what isn't (the sandbox honesty section)

The NetworkPolicy (`networkpolicy.yaml`) enforces, via kube-router on your
stock K3s:

- ✅ **LAN blocked** — the agent cannot reach `192.168.x.x`: not your router,
  not deus/nebula/legion management, not NAS admin, not other devices. *This is
  the containment that matters most.*
- ✅ Link-local / metadata (`169.254/16`) blocked.
- ✅ DNS + intra-cluster allowed (needed to function).
- ⚠️ **General internet on 443/80 is allowed** — required because `code tunnel`
  and `git clone` need out. This is IP-based, so it's "all internet on 443," not
  a domain allowlist.

Plus pod hardening: non-root (uid 1000), all capabilities dropped, no privilege
escalation, no host mounts, RuntimeDefault seccomp, and a hard memory ceiling so
a runaway process can't OOM the node.

**What this does NOT give you:** a domain allowlist like your bash sandbox
(`github.com` yes, everything else no). NetworkPolicy can't express that —
domains map to rotating CDN IPs. That's the next section.

---

## 8. Follow-up: true domain allowlist via egress proxy

When you want the bash-sandbox-grade "only these domains" containment, it's a
distinct project worth its own write-up:

1. Run a small **Squid** (or similar) in-cluster with a domain allowlist
   (`github.com`, `ghcr.io`, `pypi.org`, `crates.io`, `go.dev`, the VS Code
   tunnel domains, …).
2. Set `HTTP_PROXY`/`HTTPS_PROXY` env in the dev pod to point at it.
3. Tighten this NetworkPolicy to **block direct 443/80** and allow egress *only*
   to the proxy's ClusterIP — so the proxy is the single way out and the
   allowlist is enforced.

That inverts the model from "block the bad" (LAN) to "allow only the good"
(domains) — the real equivalent of your bash setup, and a legitimate platform
concern (egress filtering) for the portfolio. Deliberately deferred so standing
up the editor and enforcing domain policy don't tangle into one hard step.

> One gotcha to expect when you do this: `code tunnel`'s relay endpoints must be
> on the allowlist or the editor won't connect. Capture them from the pod logs
> during a working session before locking egress down.

---

## 9. Verify the containment actually works

After it's running, prove the LAN block bites (this is the satisfying part):

```bash
# should SUCCEED — internet is allowed
kubectl exec -n code-server deploy/code-server -- curl -sS -m 5 https://github.com -o /dev/null && echo "internet: OK"

# should FAIL/timeout — your LAN is blocked
kubectl exec -n code-server deploy/code-server -- curl -sS -m 5 http://192.168.1.250 -o /dev/null && echo "LAN: REACHABLE (BAD)" || echo "LAN: blocked (good)"

# should FAIL/timeout — router blocked
kubectl exec -n code-server deploy/code-server -- curl -sS -m 5 http://192.168.1.1 -o /dev/null && echo "router: REACHABLE (BAD)" || echo "router: blocked (good)"
```

If the LAN checks print "blocked (good)", the sandbox is doing its job.

---

## 10. Success criteria

- [ ] `git push` to `code-server-app` → new image in ghcr → ArgoCD syncs,
      no manual `kubectl`.
- [ ] Pod schedules on `nebula` (or `deus` if nebula is loaded), never `legion`.
- [ ] Laptop VS Code connects to the `homelab-dev` tunnel and opens the cloned
      workspace.
- [ ] LAN-block verification (§9) passes.
- [ ] Deleting the pod reschedules it (placement re-evaluated) and — with
      option A — it reconnects without re-login.

---

## 11. Where this sits in the project

This is your **second app through the pipeline** (the "prove the pattern
generalizes" next-pickup from Phase 3), and the first one that exercises
NetworkPolicy, pod security context, and scheduler affinity together — all
directly transferable platform-engineering surface. It also unblocks the AI
sandbox you wanted without a second piece of infrastructure: the cluster *is*
the sandbox.

Natural follow-ups, in rough order:
1. Option (B) or wait-for-Longhorn (C) on the workspace/auth storage question.
2. The egress-proxy domain allowlist (§8) — own write-up.
3. A second profile of this image (e.g. a GPU-tolerating variant pinned to
   `legion` for AI inference work) once you want the GTX in the loop.
```
