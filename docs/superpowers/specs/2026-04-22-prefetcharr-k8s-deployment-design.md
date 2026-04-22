# Prefetcharr Kubernetes Deployment — Design

## Goal

Deploy [prefetcharr](https://github.com/p-hueber/prefetcharr) on the OpenShift
cluster (`ocp-centralis`) as a background daemon that prefetches Sonarr episodes
for active Jellyfin playback sessions. Use a Fedora-based container consistent
with other arr-stack services. Build and publish images via a CI pipeline that
fires on every push to the user's fork of `prefetcharr`. Provide a dev
deployment that auto-rolls out and a prod deployment under manual version
control.

## Non-goals

- Exposing a web UI or HTTP endpoint (prefetcharr has none).
- Persistent storage (prefetcharr has no database; logs go to stdout).
- Routing public traffic (no Service or Route needed).
- Wiring the manifests into ArgoCD or nekohouse-gitops.
- VPN/dnscrypt egress (prefetcharr only talks to in-cluster services).

## Architecture

prefetcharr is a headless Rust daemon: one process, no ports, polls
Sonarr + Jellyfin on a configured interval. The deployment is deliberately
smaller than neighbors like prowlarr.

```
 ┌──────────────── fork of p-hueber/prefetcharr ──────────────┐
 │  Rust source · Dockerfile.prefetcharr · ci/ · k8s/ · .beads│
 └─────────────┬──────────────────────────────────────────────┘
               │ git push
               ▼
 ┌──────── GitHub webhook → EventListener (servarr ns) ───────┐
 │  CEL filter: code pushes → pipeline-ci                     │
 │  CEL filter: .beads/ pushes → pipeline-agent               │
 └─────────────┬──────────────────────────────────────────────┘
               │
               ▼
 ┌──────── pipeline-ci (servarr ns) ──────────────────────────┐
 │  1. git-clone                                              │
 │  2. podman build + push :latest + :<short-sha>             │
 │  3. oc rollout restart deployment/prefetcharr-dev          │
 └─────────────┬──────────────────────────────────────────────┘
               │
               ▼
 ┌──────── Deployments (servarr ns) ──────────────────────────┐
 │  prefetcharr-dev  image: :latest         replicas: 0       │
 │  prefetcharr      image: :<pinned-sha>   replicas: 1       │
 └─────────────┬──────────────────────────────────────────────┘
               │
               ▼
 ┌──────── In-cluster targets ────────────────────────────────┐
 │  sonarr.servarr.svc.cluster.local                          │
 │  jellyfin.jellyfin.svc.cluster.local:8096                  │
 └────────────────────────────────────────────────────────────┘
```

## Prerequisites (outside this design's scope)

- User forks `p-hueber/prefetcharr` to their own org and clones it into
  `/home/peter/workdir/prefetcharr/`.
- `bd init` has been run in the checkout to create `.beads/`.
- Two secrets exist in Doppler:
  - `PREFETCHARR_JELLYFIN_API_KEY`
  - `PREFETCHARR_SONARR_API_KEY`
- The `quay-push-secret` (dockerconfigjson for Quay) exists in the `servarr`
  namespace, matching the pattern used by ci-framework's own release pipeline.
- The `init-servarrbot-pull-secret` exists in the `servarr` namespace for
  pulling the image at runtime (already used by prowlarr).
- GitHub webhook is registered against the EventListener's Route URL.

## Component 1 — Dockerfile.prefetcharr

Multi-stage, both stages `registry.fedoraproject.org/fedora:43`.

- **Builder stage**: installs `rust`, `cargo`, `gcc`, `openssl-devel`,
  `pkgconf-pkg-config` from dnf. `COPY . /src`. Runs
  `cargo build --release --locked` against the local checkout (uses the fork's
  pinned `Cargo.lock`).
- **Runtime stage**: fresh `fedora:43`. Installs `ca-certificates` and
  `openssl`. Copies `/src/target/release/prefetcharr` from the builder to
  `/usr/local/bin/prefetcharr`. `ENTRYPOINT ["/usr/local/bin/prefetcharr"]`,
  `CMD ["--config", "/etc/prefetcharr/config.toml"]`.

Sibling `.dockerignore` excludes `target/`, `.git/`, `.beads/`.

No `USER` directive — runs as root inside the container, matching prowlarr.
Can tighten later if needed.

## Component 2 — Configuration (ExternalSecret)

Non-secret config is visible in gitops YAML; secrets (API keys) are pulled from
Doppler via ESO. This mirrors the `es-postgresql.yaml` pattern in `rhbk`.

**`k8s/externalsecret-prefetcharr-config.yaml`** produces a Kubernetes Secret
named `prefetcharr-config` with a single key `config.toml` containing the
materialized TOML:

```toml
interval = 900
log_level = "Info"
prefetch_num = 2
request_seasons = true
connection_retries = 6

[media_server]
type = "Jellyfin"
url = "http://jellyfin.jellyfin.svc.cluster.local:8096"
api_key = "<from Doppler: PREFETCHARR_JELLYFIN_API_KEY>"

[sonarr]
url = "http://sonarr.servarr.svc.cluster.local"
api_key = "<from Doppler: PREFETCHARR_SONARR_API_KEY>"
```

`log_dir` is intentionally omitted so prefetcharr logs to stderr, which K8s
captures for `kubectl logs`.

`refreshInterval: 1h` — ESO re-pulls hourly. Rotating a key in Doppler
re-materializes the Secret but the running pod still holds the old file from
its volume mount; an `oc rollout restart` picks up the new values. Acceptable
because rotation is infrequent.

Both `prefetcharr-dev` and `prefetcharr` deployments consume the same
`prefetcharr-config` Secret. If split configs become necessary later, rename
this ExternalSecret to `prefetcharr-config-prod` and add a `-dev` variant.

## Component 3 — Deployments

Two Deployments in the `servarr` namespace, identical in shape except for
name, image tag, and replica count.

| Field | `prefetcharr-dev` | `prefetcharr` |
|---|---|---|
| `replicas` | `0` | `1` |
| `image` | `quay.nekohouse.ca:8443/init/prefetcharr:latest` | `quay.nekohouse.ca:8443/init/prefetcharr:<short-sha>` |
| `imagePullPolicy` | `Always` | `Always` |
| Auto-rollout from CI | Yes | No |

Shared settings:

- Labels: `app: <name>`, `app.kubernetes.io/part-of: servarr` (matches
  prowlarr's labels).
- `imagePullSecrets: [init-servarrbot-pull-secret]`.
- Single container `prefetcharr`, args from Dockerfile CMD.
- Volume mount: `/etc/prefetcharr` (readOnly) from Secret `prefetcharr-config`,
  key `config.toml` → file `config.toml`.
- No `dnsConfig`, no sidecars, no `ports`, no PVC, no Service, no Route.
- No liveness/readiness probes — relies on the default "restart on container
  exit" behavior. prefetcharr has no HTTP endpoint to probe.

**Promotion from dev to prod:** after validating a build in dev, edit
`k8s/deployment-prefetcharr.yaml` to bump the image tag from the previous sha
to the new sha, commit, and `oc apply -k k8s/`. The gitops commit is the
audit trail.

**Duplicate-request caveat:** two prefetcharr instances against the same
Sonarr + Jellyfin both trigger searches for the same active session. Convention:
dev is normally scaled to 0. When testing a change, scale dev to 1 and prod
to 0. Leaving both running for short windows is acceptable — Sonarr dedupes
most duplicated search requests.

## Component 4 — RBAC for CI auto-rollout

**`k8s/rbac-prefetcharr-ci.yaml`** — Role + RoleBinding in the `servarr`
namespace. Scoped narrowly so CI can only touch the dev deployment:

- `Role`: `apiGroups: [apps]`, `resources: [deployments]`, `verbs: [get, patch]`,
  `resourceNames: [prefetcharr-dev]`.
- `RoleBinding`: binds the default `pipeline` ServiceAccount (used by
  openshift-pipelines for Tekton PipelineRuns) to the Role.

The prod deployment is deliberately **not** included in `resourceNames`. CI
cannot restart prod.

## Component 5 — CI pipelines

Three Tekton resources in `ci/`, modeled on ci-framework and garage-opener.

### `ci/pipeline-ci.yaml`

Three tasks:

1. **`clone`** — `taskRef` to cluster-resolved `git-clone` task
   (`openshift-pipelines` namespace). Writes source to workspace.
2. **`build-push`** — inline `taskSpec` (not `x86-build` — we need push
   functionality that `x86-build` doesn't provide). Uses
   `quay.nekohouse.ca:8443/ci-framework/x86:latest` as the runner (Fedora +
   podman). Mounts `quay-push-secret` at `/root/.docker/`. Runs:
   ```
   SHORT_SHA="${revision:0:8}"
   podman build --tls-verify=false \
     -f Dockerfile.prefetcharr \
     -t quay.nekohouse.ca:8443/init/prefetcharr:latest \
     -t quay.nekohouse.ca:8443/init/prefetcharr:$SHORT_SHA \
     .
   podman push --tls-verify=false quay.nekohouse.ca:8443/init/prefetcharr:latest
   podman push --tls-verify=false quay.nekohouse.ca:8443/init/prefetcharr:$SHORT_SHA
   ```
3. **`rollout-restart`** — inline `taskSpec` using
   `registry.redhat.io/openshift4/ose-cli:latest` (the standard `oc` CLI
   image shipped with OpenShift). Runs:
   `oc rollout restart deployment/prefetcharr-dev -n servarr`.

Runs in the `servarr` namespace with the default `pipeline` ServiceAccount.
RBAC (Component 4) grants that SA the permissions to restart the dev
deployment.

### `ci/pipeline-agent.yaml`

Copy of `garage-opener/ci/pipeline-agent.yaml` with the pipeline name changed
to `prefetcharr-agent`. Three parallel `ci-agent` task references (from
ci-framework), each cloning independently to avoid git conflicts.

### `ci/trigger.yaml`

Copy of `ci-framework/triggers/github-trigger.yaml` with:

- `<PROJECT_NAME>` → `prefetcharr`
- `<NAMESPACE>` → `servarr`

Keeps both CEL filter triggers:

- CI trigger fires when any commit touches a file outside `.beads/`.
- Agent trigger fires when any commit touches a `.beads/` file.

The agent-commits-only-touch-code guarantee in the ci-framework pattern
prevents a loop.

## File inventory

```
/home/peter/workdir/prefetcharr/             (= user's fork of p-hueber/prefetcharr)
├── Dockerfile.prefetcharr
├── .dockerignore
├── .beads/                                   (generated by `bd init`)
├── ci/
│   ├── pipeline-ci.yaml
│   ├── pipeline-agent.yaml
│   └── trigger.yaml
├── k8s/
│   ├── kustomization.yaml
│   ├── externalsecret-prefetcharr-config.yaml
│   ├── deployment-prefetcharr-dev.yaml
│   ├── deployment-prefetcharr.yaml
│   └── rbac-prefetcharr-ci.yaml
└── docs/superpowers/specs/2026-04-22-prefetcharr-k8s-deployment-design.md
```

Nothing is added to `nekohouse-gitops`. Future work (explicitly out of scope
here) will extract a separate private gitops repo for servarr-related
manifests; when that happens, `k8s/` lifts verbatim into
`components/prefetcharr/` in the new repo.

## Deliberately omitted (YAGNI)

- Service, Route, PVC, HPA — prefetcharr has no port, no state, no scalable
  workload shape.
- Liveness/readiness probes — no HTTP endpoint; restart-on-exit is sufficient.
- Prometheus ServiceMonitor, NetworkPolicy — can be added if monitoring
  requirements emerge.
- Distroless runtime image — Fedora 43 runtime is larger than necessary but
  keeps the namespace consistent. Revisit if registry storage becomes an
  issue.
- Non-root `USER` directive — prowlarr and neighbors run as root; match the
  convention.
- Agent pipeline size tuning — default 3 agents; tune when real load exists.

## Validation plan

1. `podman build -f Dockerfile.prefetcharr .` succeeds locally and produces a
   runnable image.
2. `podman run --rm <image> --help` prints the prefetcharr CLI help, confirming
   the binary is present and executable in the runtime layer.
3. `oc apply -k k8s/` succeeds against the cluster with no errors after the
   Doppler keys and `quay-push-secret` are in place.
4. `oc get secret prefetcharr-config -n servarr` shows the Secret materialized
   by ESO with key `config.toml`.
5. Scaling `prefetcharr-dev` to `1` replica produces a pod that logs
   `Initialising session monitor` (or equivalent) and begins polling.
6. A code push to the fork triggers a PipelineRun that completes all three
   tasks.
7. A `.beads/` push triggers the agent pipeline.
