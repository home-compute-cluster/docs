# Lessons: Secrets and Credentials

Secrets are where "it looks right" and "it is right" drift furthest apart, because
you usually cannot see the value you are debugging. Most of these came out of the
cert-manager work in Week 8 and the CNPG backup chase in Week 11.

---

## 1. Seal the value the consumer expects, not the one that is easiest to copy

**What happened.** Garage gives each access key both a friendly name (`cnpg-key`)
and an ID that starts with `GK`. I sealed the name. Garage answered
`No such key: cnpg-key`, and the CNPG archiver never shipped a single WAL segment.

**The lesson.** Many S3-compatible stores separate a human label from the actual
credential. Only the ID and its secret authenticate.

**What I do now.** Get the exact values from the tool that issued them
(`garage key info --show-secret <key>`) and seal those, rather than typing from
memory.

## 2. Name secrets after their role, not the product behind them

**What happened.** The backup credentials started life as `minio-creds`. When I
switched from MinIO to Garage I renamed the secret to `backup-store-creds`, but the
CNPG `Cluster` still referenced `minio-creds`. Two of the five failures in that
chase came from the stale name.

**The lesson.** A secret named after an implementation becomes wrong the moment the
implementation changes, and then someone renames it and misses a reference.

**What I do now.** Name secrets by what they are for (`backup-store-creds`,
`grafana-admin`). After any rename, search the whole config repo for the old name
before pushing.

## 3. Kubernetes object names only allow lowercase letters, digits, hyphens and dots

**What happened.** While fixing the reference above, I typed `backup_store_creds`.
Kubernetes names follow the DNS subdomain rules from RFC 1123, which do not allow
underscores, so that reference could never match anything.

**The lesson.** An invalid name in a reference does not always fail loudly. It can
just look like a missing object.

**What I do now.** Hyphens only, lowercase only, in every name.

## 4. A SealedSecret is locked to its namespace and name

**What happened.** The Cloudflare API token for cert-manager's DNS-01 challenge was
sealed for the wrong namespace, so the controller could not decrypt it and the
challenge kept failing. The confusing part is that the token (the issuer's input)
has to live in the `cert-manager` namespace, because a ClusterIssuer looks for its
credentials in cert-manager's own namespace, while the certificate it produces for
Traefik's TLSStore lives in `kube-system`.

**The lesson.** Sealed Secrets encrypts against namespace and name by default.
Seal for the wrong one and decryption fails, full stop.

**What I do now.** Always pass `--namespace` explicitly to `kubeseal`, and write
down which namespace each credential is read from before sealing it.

## 5. Generating a SealedSecret is not the same as applying it

**What happened.** At one point in Week 8 the SealedSecret file existed, looked
right, and had never been applied or committed where ArgoCD would see it. I lost
about an hour to that.

**What I do now.** After sealing, check that the resulting Secret exists in the
cluster: `kubectl get secret <name> -n <ns>`.

## 6. Back up the Sealed Secrets controller key, outside the cluster and outside Git

**Why it matters.** Every SealedSecret in the repo is encrypted for the
controller's private key. If the cluster is rebuilt and that key is gone, every
sealed secret becomes unreadable at once.

**What I do now.** Export it after any key rotation and keep it somewhere that is
neither the cluster nor the config repo:

```bash
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key.yaml
```

This was done in Week 10 and is one of the few items on the gap audit that is
genuinely closed.

## 7. Credentials leak in through side doors you did not configure

**What happened.** In Week 5 the code-server dev container could `git push` to
GitHub with no token, no stored credentials and a clean remote URL. The VS Code
tunnel server injects `GIT_ASKPASS` and `VSCODE_GIT_*` environment variables into
shells it starts, which brokers git credentials through my signed-in editor
session. The agent running inside had account-wide write access that I never
granted it.

**The lesson.** Network policies and filesystem scoping do nothing against a
credential that arrives through an environment variable. Also, those variables were
injected after the entrypoint ran, so unsetting them in the entrypoint did not help.

**What I do now.** Two independent fixes, both needed. First, a fine-grained PAT
scoped to a single repository, which is the fix that actually limits damage.
Second, closing every shell entry point: `/etc/profile.d` for login shells,
`BASH_ENV` for non-interactive `bash -c`, and `git config --system core.askPass ""`
as a fallback. Test with the variables removed to prove it:
`GIT_ASKPASS= git -c core.askPass= ls-remote <repo>`.

## 8. An error message about caching can really be about credentials

**What happened.** The CNPG archiver logged `cache miss` for days. It was not a
caching bug. The archiver builds its environment from the credential secret, and
when the secret lookup fails, the cache has nothing to build from.

**The lesson.** Read an error as a symptom of whatever sits upstream of it. The
backup chase went secret resolution, then credential validity, then S3 auth, then
region negotiation, and each fix exposed the next layer down.

**What I do now.** When a component reports a vague internal error, check the
inputs it depends on (secrets, config, endpoints) before assuming the component
itself is broken.
