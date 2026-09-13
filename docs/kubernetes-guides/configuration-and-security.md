# Configuration, Secrets and Security

*Part of the [K3s / Kubernetes working reference](k3s-kubernetes-guide.md).*

Getting configuration and credentials into Pods, and the basics of controlling who and what can do things in the cluster.

## Configuration and secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config, namespace: web }
data:
  UPSTREAM_HOST: "backend.web.svc.cluster.local"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata: { name: app-secret, namespace: web }
type: Opaque
stringData:                 # plain text in, stored base64 encoded
  api_token: "s3cr3t"
```

```yaml
    containers:
      - name: app
        envFrom:
          - configMapRef: { name: app-config }
        env:
          - name: API_TOKEN
            valueFrom:
              secretKeyRef: { name: app-secret, key: api_token }
```

**Secrets are base64 encoded, not encrypted.** Anyone who can read Secrets in a
namespace can decode them. Two separate problems need solving:

- **In Git:** plain Secrets cannot be committed. Sealed Secrets, SOPS or External
  Secrets Operator solve this. This cluster uses Sealed Secrets, where only the
  in-cluster controller can decrypt, and the controller's key must be backed up
  outside the cluster.
- **At rest in the datastore:** K3s can encrypt Secrets in etcd with
  `--secrets-encryption`, which is off by default.

Environment variables from a Secret are read when the container starts. Changing the
Secret does not update a running container's environment; restart the Deployment.
Values mounted as files are updated eventually.

---

## Security basics

### RBAC

Access to the API is controlled with four objects:

| Object | Scope | Purpose |
|---|---|---|
| **Role** | one namespace | a set of allowed verbs on resources |
| **ClusterRole** | cluster-wide | same, for cluster-scoped resources or for reuse across namespaces |
| **RoleBinding** | one namespace | grants a Role (or a ClusterRole, limited to that namespace) to subjects |
| **ClusterRoleBinding** | cluster-wide | grants a ClusterRole everywhere |

Permissions only add up; **there are no deny rules**. Subjects are users, groups or
**ServiceAccounts**, which is how workloads get API access.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: web
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: web
subjects:
  - kind: ServiceAccount
    name: dashboard
    namespace: web
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Every Pod runs as a ServiceAccount (`default` if none is set). Give workloads their
own ServiceAccount with only what they need, and set
`automountServiceAccountToken: false` for Pods that never call the API. Binding a
dashboard to `cluster-admin` is convenient and gives anyone who reaches that dashboard
full control of the cluster.

### Pod Security Admission

Kubernetes has three built-in Pod Security Standards, enforced per namespace with
labels:

- **privileged**: no restrictions.
- **baseline**: blocks known privilege escalations (host namespaces, privileged
  containers and similar).
- **restricted**: current hardening best practice (non-root, dropped capabilities,
  seccomp).

Each level can be applied in three modes: `enforce` rejects violating Pods, `audit`
records them in the audit log, `warn` shows a warning to the user.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
```

### securityContext

The per-Pod and per-container settings that most workloads should have:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities: { drop: ["ALL"] }
```

Note that `readOnlyRootFilesystem` is a **container** field. Putting it in the Pod
level `securityContext` is invalid, which server-side apply will reject.

---

## Sources

- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [k3s server CLI reference](https://docs.k3s.io/cli/server)
