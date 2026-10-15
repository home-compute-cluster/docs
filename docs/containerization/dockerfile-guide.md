# Dockerfile Reference Guide

A practical reference for writing Dockerfiles across common use cases. The goal
throughout is the same: produce a **small, reproducible, cache-friendly** image,
whether you're building for a CI pipeline, a registry push, or a Kubernetes
cluster.

---

## 1. How a Dockerfile actually works (the mental model)

A Dockerfile is a sequence of instructions. Each instruction that changes the
filesystem creates a **layer**, an immutable diff stacked on the previous one.
The final image is all layers stacked together.

Two consequences drive almost every best practice:

- **Layers are cached.** On rebuild, Docker reuses a cached layer if the
  instruction *and all instructions before it* are unchanged. The moment one
  instruction's inputs change, that layer and everything after it rebuild. This
  is why instruction *ordering* matters so much: put rarely-changing things
  (dependency installs) before frequently-changing things (your source code).
- **Layers only add, never subtract.** Deleting a file in a later layer doesn't
  shrink the image; the bytes still exist in the earlier layer. To actually
  remove something, it must be cleaned up *within the same instruction* that
  created it (or avoided via multi-stage builds).

A build runs in a temporary container per instruction; the result is committed
as a layer, the container is discarded, and the next instruction starts from
that layer.

---

## 2. The instructions, one by one

### `FROM`: the base image

Starts every build (or every *stage* of a multi-stage build). Defines the
starting filesystem and OS userland.

```dockerfile
FROM nginx:1.27-alpine
FROM golang:1.23 AS builder        # named stage, for multi-stage
FROM scratch                       # empty, literally nothing
```

- **Pin a specific tag**, never bare `latest`. `nginx:1.27-alpine` is
  reproducible; `nginx` silently changes under you and breaks any "exact build"
  guarantee your pipeline depends on.
- For maximum reproducibility, pin by digest: `FROM nginx:1.27-alpine@sha256:...`.
- `-alpine`, `-slim`, and `scratch` variants are progressively smaller. More on
  base-image choice in section 5.

### `RUN`: execute a command at build time

Runs a shell command and commits the result as a layer. This is where you
install packages, compile code, create directories.

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

Two forms:

- **Shell form:** `RUN apt-get update`, runs via `/bin/sh -c`. Supports `&&`,
  pipes, variable expansion.
- **Exec form:** `RUN ["executable", "arg1", "arg2"]`, no shell, no `&&`. Rare
  for `RUN`; common for `CMD`/`ENTRYPOINT`.

Chain related commands with `&&` and clean up in the same `RUN` (see section 3), since
a separate `RUN rm ...` can't shrink the earlier layer.

### `COPY`: copy files from build context into the image

```dockerfile
COPY index.html /usr/share/nginx/html/index.html
COPY go.mod go.sum ./          # copy just deps first (cache trick, section 4)
COPY . .                       # copy everything
```

- The **build context** is the directory you pass to `docker build` (usually
  `.`). `COPY` can only see files inside it; you can't `COPY /etc/passwd`.
- Prefer `COPY` over `ADD` for plain files. `COPY` does exactly one thing.

### `ADD`: copy, but with extra magic

`ADD` does what `COPY` does, plus: auto-extracts local tarballs, and can fetch
URLs. The magic is usually a footgun.

```dockerfile
ADD project.tar.gz /app/       # auto-extracts the tarball
```

- Use `ADD` **only** when you specifically want tarball auto-extraction.
- For remote files, prefer `RUN curl`/`wget` (explicit, lets you verify
  checksums and clean up in one layer). Don't use `ADD <url>`.

### `WORKDIR`: set the working directory

```dockerfile
WORKDIR /app
```

Sets the cwd for subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD`. Creates
the directory if absent. Use it instead of `RUN cd /app`: `cd` in a `RUN`
doesn't persist to the next instruction (each `RUN` is a fresh shell).

### `ENV`: set environment variables

```dockerfile
ENV APP_PORT=8080 \
    LOG_LEVEL=info
```

Persists into the running container (visible to your process). Use for runtime
config defaults. For build-time-only values, prefer `ARG`.

### `ARG`: build-time variables

```dockerfile
ARG GO_VERSION=1.23
FROM golang:${GO_VERSION}

ARG BUILD_ID                   # passed via --build-arg BUILD_ID=...
RUN echo "building $BUILD_ID"
```

- Only exists during the build; **not** present in the running container.
- An `ARG` before the first `FROM` is special: it can parameterize the base
  image, but is out of scope for stages unless re-declared.
- Don't pass secrets via `ARG`; they're visible in image history. Use
  `RUN --mount=type=secret` instead (section 8).

### `EXPOSE`: document the listening port

```dockerfile
EXPOSE 80
```

Pure documentation/metadata. It does **not** publish the port; that's
`-p` at run time, or a Kubernetes Service. Worth including so the image is
self-describing (and whoever wires up ingress knows the port).

### `CMD`: default command

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

The default process the container runs. Overridable at run time
(`docker run image <other-command>`). Prefer **exec form** (JSON array) so your
process is PID 1 and receives signals (clean `SIGTERM` shutdown, matters for
Kubernetes pod termination). Shell form (`CMD nginx -g ...`) wraps your process
in `/bin/sh`, which swallows signals.

### `ENTRYPOINT`: the fixed executable

```dockerfile
ENTRYPOINT ["/app/server"]
CMD ["serve"]                  # default args, appended to ENTRYPOINT
```

`ENTRYPOINT` sets the executable that always runs; `CMD` provides default
*arguments* to it. `docker run image migrate` would run `/app/server migrate`.
This `ENTRYPOINT` + `CMD` pairing is the idiomatic pattern for a single binary
that exposes multiple subcommands (e.g. a server with `serve` / `migrate` /
`health` verbs).

### The rest, briefly

- **`USER`**: drop to a non-root user. `USER 1000` or `USER appuser`. Do this
  for anything you expose; running as root in-container is a needless risk.
- **`VOLUME`**: declares a mount point for external data. In Kubernetes you
  manage this via PVCs instead, so you rarely need `VOLUME` in cluster images.
- **`HEALTHCHECK`**: a command Docker runs to test liveness. In Kubernetes the
  kubelet's liveness/readiness probes supersede this, so it's often omitted for
  cluster workloads.
- **`LABEL`**: metadata key/values. Useful for
  `org.opencontainers.image.source` so a registry (e.g. ghcr.io) links the
  package back to its source repo.
- **`SHELL`**: change the shell used by shell-form `RUN` (e.g. to `bash`).
  Rarely needed.
- **`ONBUILD`**: defers an instruction to child images. Niche; mostly avoid.
- **`STOPSIGNAL`**: overrides the signal sent to stop the container. Rarely
  needed.

---

## 3. The cleanup rule (why images bloat)

Because layers only add, you must clean up **inside the same `RUN`**:

```dockerfile
# GOOD: apt cache removed in the same layer it was created
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

```dockerfile
# BAD: the cache bytes are already committed to an earlier layer;
# this rm shrinks nothing.
RUN apt-get update && apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

Same logic for: `pip install --no-cache-dir`, `npm ci && npm cache clean
--force`, `go build` caches, downloaded tarballs you extracted then deleted.
`--no-install-recommends` (apt) avoids pulling optional extras you don't need.

---

## 4. Layer caching: order for fast rebuilds

The single highest-leverage technique. Copy and install **dependencies before
source code**, because deps change rarely and source changes every commit:

```dockerfile
# Node example
WORKDIR /app
COPY package.json package-lock.json ./   # changes rarely
RUN npm ci                               # cached unless lockfile changes
COPY . .                                 # changes every push
RUN npm run build
```

```dockerfile
# Go example
WORKDIR /src
COPY go.mod go.sum ./                     # changes rarely
RUN go mod download                       # cached unless go.mod/sum change
COPY . .                                  # changes every push
RUN go build -o /app/server ./cmd/server
```

If you wrote `COPY . .` *before* the dependency install, every one-character
source edit would invalidate the install layer and re-download all dependencies,
turning a 5-second rebuild into minutes. On a CI runner that's build minutes
and queue time billed directly.

---

## 5. Choosing a base image

| Base | Size | When |
|------|------|------|
| Full distro (`debian`, `ubuntu`, `golang`) | Large (100s of MB) | Build stages; when you need a full toolchain |
| `-slim` (`python:3.12-slim`, `debian:slim`) | Medium | Runtime when you need glibc + a package manager but not the full distro |
| `-alpine` (`nginx:1.27-alpine`) | Small (~5-50 MB) | Runtime; musl libc. Watch for musl-vs-glibc quirks with some binaries/wheels |
| `gcr.io/distroless/*` | Small, no shell | Runtime for compiled binaries; minimal attack surface, no shell to exec into |
| `scratch` | Empty | Fully static binaries (typical static Go); nothing but your binary |

Trade-off: smaller bases pull faster (pods start sooner on a new node), have fewer CVEs, but give you less to debug with (no shell in
`scratch`/distroless). For a static Go service, `scratch` or `distroless/static`
is ideal. For nginx serving static files, `nginx:1.27-alpine` is the standard.

---

## 6. Multi-stage builds (the big one)

Build in a fat image with the full toolchain; ship only the artifact in a tiny
image. This is the most important pattern for compiled languages.

### Go service to a tiny final image

```dockerfile
# ---- build stage: has the Go toolchain ----
FROM golang:1.23 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# CGO_ENABLED=0 makes a fully static binary that runs on scratch
RUN CGO_ENABLED=0 GOOS=linux go build -o /out/server ./cmd/server

# ---- final stage: just the binary ----
FROM scratch
COPY --from=builder /out/server /server
# bring CA certs if the binary makes outbound HTTPS calls
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/server"]
CMD ["serve"]
```

The `golang:1.23` toolchain (hundreds of MB) never ships. The final image is
your binary plus certs, often under 20 MB. `COPY --from=builder` is the key:
it pulls a file *out* of an earlier stage into the final one.

### Node app to built assets only

```dockerfile
FROM node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build              # produces ./dist

FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

A static-site generator (Hugo, Astro, plain webpack/Vite) fits this exactly: a
`build` stage runs the generator, the final stage is nginx serving the output.

### Python app to a slim runtime

```dockerfile
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
COPY --from=build /install /usr/local
COPY . /app
WORKDIR /app
EXPOSE 8000
CMD ["python", "-m", "myapp"]
```

---

## 7. Worked examples by use case

### Static site (nginx serving files)

```dockerfile
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
# custom nginx config if you need routing/headers:
# COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

`daemon off;` is required: nginx daemonizes by default, which would make PID 1
exit and kill the container. This keeps it in the foreground.

### Go binary with subcommands

Use the multi-stage Go pattern in section 6. The `ENTRYPOINT ["/server"]` + `CMD
["serve"]` shape means:

- default run: `/server serve`
- a one-off Job that overrides `CMD` with `["migrate"]` runs `/server migrate`

This maps cleanly onto any single binary that exposes multiple verbs
(`serve` / `migrate` / `health`), which is a common shape for a service that
ships its own admin commands.

### A compiled tool that needs a shell for debugging

If `scratch` is too austere (you want to `kubectl exec` in and poke around),
use `distroless` with a debug variant, or `alpine`:

```dockerfile
FROM golang:1.23 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM alpine:3.20
RUN apk add --no-cache ca-certificates
COPY --from=builder /out/app /usr/local/bin/app
USER 1000
ENTRYPOINT ["app"]
```

`alpine` gives you a shell and `apk` while staying small; a good default while
you're still iterating on and debugging a service, before you tighten it down to
`scratch`/distroless for production.

---

## 8. Security and correctness checklist

- **Run as non-root.** Add `USER` once the process doesn't need root. Create a
  user in the build if needed: `RUN adduser -D -u 1000 app` then `USER app`.
- **Pin base image tags** (and ideally digests): reproducibility.
- **Don't bake secrets in.** Not via `ENV`, not via `ARG`, not `COPY .env`.
  They persist in image layers/history. Use Kubernetes Secrets at runtime, or
  BuildKit secret mounts at build time:
  ```dockerfile
  RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
  ```
- **Use a `.dockerignore`.** Keeps junk (`.git`, `node_modules`, `*.md`, local
  env files) out of the build context: faster builds, smaller context, no
  accidental secret leakage:
  ```
  .git
  node_modules
  *.log
  .env
  ```
- **Exec form for `CMD`/`ENTRYPOINT`** so signals reach your process (clean pod
  termination on Kubernetes).
- **Minimal final image**: fewer packages, fewer CVEs, faster pulls.

---

## 9. Quick reference table

| Instruction | Purpose | Key gotcha |
|-------------|---------|-----------|
| `FROM` | Base image / start a stage | Pin the tag; never bare `latest` |
| `RUN` | Execute at build time, creating a new layer | Clean up in the *same* `RUN` |
| `COPY` | Copy context files in | Source must be inside build context |
| `ADD` | Copy + tar-extract / URL fetch | Avoid unless you want tar extraction |
| `WORKDIR` | Set cwd | Use instead of `RUN cd` (cd doesn't persist) |
| `ENV` | Runtime env var | Persists into the container |
| `ARG` | Build-time var | Not in the running container; not for secrets |
| `EXPOSE` | Document a port | Doesn't publish anything |
| `CMD` | Default command/args | Exec form for signals; overridable |
| `ENTRYPOINT` | Fixed executable | Pair with `CMD` for default args |
| `USER` | Drop privileges | Set before the process runs |
| `LABEL` | Metadata | `image.source` links registry package to repo |
| `HEALTHCHECK` | Liveness probe | K8s probes usually supersede |
| `VOLUME` | External mount point | Use PVCs in K8s instead |

---

## 10. How a Dockerfile fits a build pipeline

A typical CI step builds from the repo root: `docker build -t $IMAGE:$SHA -t
$IMAGE:latest .`. So the Dockerfile lives at the repo root, sees that directory
as its build context, and should:

1. **Pin its base image**: the image content becomes as reproducible as the SHA
   tag, so a given commit always produces a functionally identical image.
2. **Order deps-before-source** so runner build minutes stay low.
3. **Multi-stage down to a small final image** so the pull onto each node is
   quick and pods start sooner.
4. **Add `LABEL org.opencontainers.image.source=https://github.com/<owner>/<repo>`**
   so the registry package auto-links back to the source repo.

Off-the-shelf images (databases, Grafana, media servers, etc.) have no
Dockerfile at all; they skip the build step entirely and are referenced
directly by their upstream image tag in your deployment manifests.
