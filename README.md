# DevOps/GitOps POC

A local, end-to-end **DevOps + DevSecOps + GitOps** proof of concept: three Node.js microservices, containerized, security-scanned, continuously delivered through GitHub Actions and Docker Hub, and deployed to Kubernetes via Argo CD — with Prometheus/Grafana observability and deliberately tested failure recovery.

This repository is the **GitOps / deployment side** — Helm charts, Kubernetes configuration, Argo CD, and the monitoring stack. Application source lives in [`devops-poc-app`](https://github.com/trushang-dev/devops-poc-app).

![Architecture: CI/CD + GitOps + Kubernetes + Observability](docs/architecture.png)

For architectural reasoning and diagrams, see [ARCHITECTURE.md](ARCHITECTURE.md). For the milestone-by-milestone build story, see [PROJECT_JOURNEY.md](PROJECT_JOURNEY.md).

## Overview

This project exists to demonstrate how a production-style software delivery pipeline actually fits together — Git, CI, a container registry, GitOps, Kubernetes, and observability — without requiring AWS or any paid cloud infrastructure. Everything runs locally against Minikube. The application itself (three small Express APIs) is deliberately simple; the pipeline and operational practices around it are the point.

Every stage described below was implemented and then verified against the live system, not just configured — real load tests, real drift injected and reconciled, real rollbacks performed both the Kubernetes-native way and the GitOps way, and real failures induced and recovered from. Milestone completion reports with exact commands, timestamps, and log evidence exist for the higher-risk claims (Docker Hub/CI, Argo CD, deployment strategy, failure testing); this README summarizes them without repeating the raw evidence.

## Architecture Overview

```text
Developer
   │  git push / PR
   ▼
devops-poc-app (GitHub)
   │
   ▼
GitHub Actions  ──  npm test → Trivy filesystem scan
   │  (release/* branches only, beyond this point)
   ▼
Docker build  →  Trivy image scan  →  Docker Hub push
   │
   ▼
devops-poc-gitops (main) — Helm values.yaml image tag bumped by CI
   │
   ▼
Argo CD  — watches devops-poc-gitops@main, reconciles automatically
   │
   ▼
Kubernetes (Minikube) — devops-poc namespace
   │
   ▼
Prometheus / Grafana / OpenTelemetry Collector — devops-monitoring namespace
```

GitHub Actions never touches Kubernetes. Its last responsibility is a Git commit to the GitOps repository; Argo CD is the only component that ever applies a change to the cluster. This split is the core security property of the design: a compromised CI pipeline can write to a Git repository, nothing more.

## Services

| Service | Port | Business endpoint | Also exposes |
|---|---:|---|---|
| `user-service` | 3001 | `GET /users` — in-memory user list | `GET /health`, `GET /version`, `GET /metrics` |
| `product-service` | 3002 | `GET /products` — in-memory product list | `GET /health`, `GET /metrics` |
| `order-service` | 3003 | `GET /orders` — in-memory order list | `GET /health`, `GET /metrics` |

Each service is an independent Express app with no database — an in-memory array stands in for persistence. There is no cross-service communication; they exist to give the pipeline three real, independently deployable artifacts rather than one. Each is instrumented with `prom-client`: an `http_requests_total` counter and an `http_request_duration_seconds` histogram (both labeled `method`/`route`/`status_code`), plus Node.js's own default process metrics, all exposed on `/metrics` in Prometheus text format. `/health` is an unconditional 200 responder used by both Kubernetes probes and Ingress health routes.

## Technology Stack

Only what is actually deployed and running:

| Layer | Technology |
|---|---|
| Application | Node.js 22, Express, `prom-client` |
| Containerization | Docker (`node:22-alpine`, non-root `node` user) |
| CI | GitHub Actions |
| Security scanning | Trivy (filesystem, image, and Kubernetes/Helm config scans) |
| Registry | Docker Hub |
| Packaging | Helm 3 |
| GitOps / reconciliation | Argo CD |
| Orchestration | Kubernetes (Minikube), `autoscaling/v2` HPA |
| Ingress | NGINX Ingress Controller (path-based routing) |
| Metrics | Prometheus (via `kube-prometheus-stack`), `kube-state-metrics`, Node Exporter |
| Telemetry gateway | OpenTelemetry Collector (`otel/opentelemetry-collector-contrib`) — deployed, currently idle (see [Known Limitations](#known-limitations)) |
| Dashboards | Grafana, provisioned as code |

PostgreSQL was in the original project scope but was **never implemented** — see [Known Limitations](#known-limitations).

## Repository Structure

```text
devops-poc-app                          devops-poc-gitops (this repo)
├── services/                           ├── kubernetes/        (raw manifests, pre-Helm reference)
│   ├── user-service/                   ├── helm/
│   ├── product-service/                │   ├── microservices/ (the live-managed chart)
│   └── order-service/                  │   └── monitoring/    (kube-prometheus-stack + OTel values)
├── .github/workflows/ci.yml            ├── argocd/             (Application + Ingress)
└── docker-compose.yml                  └── docs/               (milestone build notes)
```

`devops-poc-app` owns the application lifecycle: source, tests, Dockerfiles, CI. `devops-poc-gitops` owns the deployment lifecycle: what should be running, and how. Helm (release name `devops-poc`) manages the live cluster resources; the raw `kubernetes/` manifests remain only as a historical record of the pre-Helm milestones.

## CI/CD Pipeline

Defined in `devops-poc-app/.github/workflows/ci.yml`, one matrix job per service. Two distinct pipelines share the same workflow file, split by trigger:

**Validation pipeline** — every push to `develop` and every pull request into it:
```text
npm ci → npm test (node:test, health + metrics smoke tests) → Trivy filesystem scan
                                                                (full report, then CRITICAL-only gate)
```
No image is built, nothing is published, nothing is deployed. This is intentional: `develop` is an integration branch, not a release.

**Release pipeline** — only a push to a `release/*` branch runs the steps above *and*:
```text
Docker build (tagged SemVer + Git SHA) → Trivy image scan (full report, then CRITICAL-only gate)
  → Docker Hub push (both tags) → update-gitops job
```
The `update-gitops` job checks out `devops-poc-gitops@main` with a repo-scoped `GITOPS_REPO_TOKEN`, bumps `helm/microservices/values.yaml`'s `image.repository`/`image.tag` with `yq`, and commits/pushes directly to `main` only if the value actually changed — no PR, no manual step. Docker Hub credentials (`DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN`) and `GITOPS_REPO_TOKEN` are GitHub Actions secrets, never code.

**Image tagging:** every image gets two tags — a SemVer tag (e.g. `v1.0.0`, the one GitOps and Argo CD actually deploy) and the Git commit SHA (kept for exact source traceability, never itself deployed). `latest` is never used.

**Why the split matters:** a `pull_request` run — including one from a fork — never has access to `DOCKERHUB_*`/`GITOPS_REPO_TOKEN`, and even an ordinary `develop` push builds nothing. Only a deliberate release-branch push can produce and publish an artifact.

## GitOps Workflow

```text
Git (devops-poc-gitops@main)  ← desired state
        │  watched continuously
        ▼
     Argo CD  ── Application "devops-poc", namespace argocd
        │  renders
        ▼
  Helm (helm/microservices)
        │  applies
        ▼
   Kubernetes (devops-poc namespace)
```

Argo CD's `devops-poc` Application runs with `syncPolicy.automated: {prune: true, selfHeal: true}`, with one deliberate exception: `ignoreDifferences` excludes `Deployment.spec.replicas`, so the HPA can scale without Argo CD reverting it back to the chart's static replica count on the next reconciliation.

**Drift detection and self-healing — real, measured evidence (not inferred from the config alone):** an out-of-band `kubectl set env` change to `user-service` was detected by the Argo CD controller (`Synced -> OutOfSync`, logged with a timestamp) and automatically reverted (`OutOfSync -> Synced`) **4 seconds later**, confirmed independently via a new `ReplicaSet` appearing and being scaled back to zero. No manual remediation was involved. This was one measured occurrence, not a claim about worst-case or guaranteed timing — see `M13_COMPLETION_REPORT.md` for the full log trail.

**Git-based rollback — also directly verified:** a real image-tag change was merged to `main` via a normal feature-branch/PR flow and deployed automatically; a `git revert` (not `reset`, not a manual `kubectl` command) on a second PR then triggered Argo CD to automatically restore the prior state. Both transitions were confirmed via controller logs and live pod images, not assumed from `selfHeal: true`.

## Kubernetes Deployment

All three services share the same Helm-templated shape (`helm/microservices/templates/deployment.yaml`, driven by one shared `values.yaml` block — not per-service overrides):

| Aspect | Configuration |
|---|---|
| Replicas | 2 (baseline), scaled 2→3 by HPA under load |
| Strategy | `RollingUpdate`, `maxSurge: 1`, `maxUnavailable: 1` |
| Readiness probe | `GET /health`, `initialDelaySeconds: 5`, `periodSeconds: 10` |
| Liveness probe | `GET /health`, `initialDelaySeconds: 15`, `periodSeconds: 20` |
| Resources | requests `50m` CPU / `64Mi` memory; limits `200m` CPU / `128Mi` memory |
| HPA | `autoscaling/v2`, `minReplicas: 2`, `maxReplicas: 3`, target `70%` CPU |

(Probe `failureThreshold`/`timeoutSeconds` are Kubernetes' own defaults — 3 and 1s respectively — since the chart doesn't override them.)

**Deployment → ReplicaSet → Pod** is the standard Kubernetes chain: the Deployment declares desired state (image, replica count, strategy); the ReplicaSet it owns ensures that many Pods exist at all times, replacing any that disappear; each Pod runs one container. A `ClusterIP` Service per service provides a stable DNS name/IP in front of the Pods, and only Pods currently passing their readiness probe appear in that Service's endpoint list.

The HPA was verified with real load (`ab`, 150 concurrency / 50k requests against `user-service` through the Ingress): CPU rose past the 70% target, a third pod was created within about a minute, load stopped, CPU dropped immediately but replica count held at 3 for the ~5-minute default downscale-stabilization window, then scaled back to 2 with no manual intervention.

## Deployment and Rollback

Two genuinely different rollback mechanisms exist in this project, both demonstrated live on `user-service`:

| | Kubernetes-native (`kubectl rollout undo`) | GitOps-native (`git revert`) |
|---|---|---|
| Source of truth | The Deployment's own `ReplicaSet` revision history, in-cluster | Git commit history (`devops-poc-gitops@main`) |
| Requires Argo CD | No | Yes — Argo CD is what applies the reverted state |
| Leaves a Git audit trail | No | Yes — a new, reviewable commit |
| Best for | A fast, local, in-cluster undo while actively debugging | The durable, reviewable record of "this is now our desired state" |

Because `selfHeal: true` would otherwise revert any `kubectl`-driven change within seconds (as directly measured in the drift test above), demonstrating the Kubernetes-native path required **temporarily** pausing Argo CD's automated sync (`kubectl patch application devops-poc ... syncPolicy.automated=null`) for the duration of that one demo, then restoring the exact original policy immediately after — a reversible, in-cluster pause, not a change to Argo CD's configuration.

**Rolling update, sampled live:** `user-service` was moved from `v1.0.0` to a second, already-proven-good image. Availability was sampled every 2 seconds throughout; at every single sample, at least one pod reported `Ready: true` — direct confirmation that `maxUnavailable: 1` kept the Service available throughout the rollout. `kubectl rollout undo` then returned it to `v1.0.0`, confirmed by both pod images and a clean new rollout revision. After Argo CD's sync was restored, it reported `Synced`/`Healthy` with **no drift left to reconcile** — proof the Kubernetes-native rollback had landed the cluster exactly where Git already said it should be.

## Failure Recovery

Four failure scenarios were scoped; each is handled by a **different** mechanism, deliberately not collapsed into one generic "self-healing" story:

| Scenario | Mechanism responsible | What was observed |
|---|---|---|
| Pod deleted directly | **ReplicaSet controller** (not Argo CD) | A replacement pod was created within seconds (`SuccessfulCreate` event), reaching `Ready` in ~16s. Argo CD has no role here — it reconciles the Deployment's *spec*, not which specific Pod object currently exists. |
| Readiness failure | **Kubelet + Service endpoint controller** | The real Node process was frozen (`kill -STOP`) inside a running container. ~32s later the pod was marked `Ready: false` and, simultaneously, removed from the Service's endpoint list — traffic stopped being routed to it. The container itself was never touched. |
| Liveness failure | **Kubelet** | The same frozen process later failed its liveness probe; ~101s after the freeze, the kubelet killed and recreated the container (restart count 0→1). The new process passed both probes and rejoined the Service on its own. |
| Bad image / rollback | **ReplicaSet history / Argo CD** (already covered) | Demonstrated earlier in `ImagePullBackOff → kubectl rollout undo` and in the rollback comparison above — not re-run as a separate test. |
| GitOps drift | **Argo CD self-heal** (already covered) | Demonstrated in the [GitOps Workflow](#gitops-workflow) section above — not re-run as a separate test. |

Neither the pod-deletion nor the process-freeze test required pausing Argo CD: both are invisible to its spec-level diffing (deleting a Pod or signaling a process inside a container never touches the Deployment/Pod specification Argo tracks). The `devops-poc` Application stayed `Synced` throughout both, with a brief expected `Progressing` blip during the container restart that settled back to `Healthy` on its own.

## Observability

Deployed in its own `devops-monitoring` namespace via the community `kube-prometheus-stack` chart (Prometheus + Grafana + `kube-state-metrics` + Node Exporter + Prometheus Operator) plus a separately-installed OpenTelemetry Collector chart.

- **Node-level metrics** (CPU, memory, disk, network) — Node Exporter.
- **Kubernetes object-state metrics** (deployment replicas, HPA state, pod restarts/readiness) — `kube-state-metrics` and cAdvisor/kubelet.
- **Application metrics** — each service's `/metrics` is scraped by a dedicated per-service `PodMonitor`, not a `ServiceMonitor`. This was a deliberate fix: scraping the Service's `ClusterIP` let `kube-proxy` load-balance each 15-second scrape across both replicas, producing a non-monotonic counter (a real bug caught while building this — two `order-service` pods once reported 35 and 31 requests respectively from a single "counter"). Scraping each Pod directly by IP resolves this.
- **OpenTelemetry Collector** — deployed as a forward-looking OTLP gateway. It previously *also* scraped application `/metrics` directly, which duplicated every metric under a second `job` label and badly distorted rate calculations (confirmed ~120× the correct rate during an audit); that scrape config has been removed. The Collector's `otlp` receiver remains configured but **currently receives nothing** — the application is instrumented with `prom-client` (Prometheus format), not an OTel SDK, so there is no active OTel data path today.
- **Grafana** — a single "DevOps POC Overview" dashboard, provisioned as code (a labeled ConfigMap plus the chart's dashboard sidecar, no manual import), with 15 panels across four rows: node metrics, Kubernetes object metrics, per-service request rate/route/status, and p95 latency + error rate + restarts + readiness.

Verified end-to-end with a real `ab -n 10000 -c 20` run against `user-service`: 0 failed requests, the request counter increased by exactly 10,000, the latency histogram recorded real values, and the HPA scaled `user-service` 2→3 while `product-service`/`order-service` correctly stayed at 2/2.

## Milestone Journey

Authoritative status per `PROJECT_SCOPE.md` §20. See [PROJECT_JOURNEY.md](PROJECT_JOURNEY.md) for the full narrative.

| ID | Milestone | Status |
|---|---|---|
| M0 | Project Foundation | ✅ Completed |
| M1 | Node.js Microservices | ✅ Completed |
| M2 | Git Workflow | ✅ Completed |
| M3 | Docker | ✅ Completed |
| M4 | Local Kubernetes | ✅ Completed |
| M5 | Kubernetes Networking (Ingress) | ✅ Completed |
| M6 | Configuration (ConfigMap/Secret) | ✅ Completed |
| M7 | Reliability (probes, resources, HPA) | ✅ Completed |
| M8 | Helm | ✅ Completed |
| M9 | Security / Trivy | ✅ Completed |
| M10 | CI Pipeline | ✅ Completed |
| M11 | Docker Hub | ✅ Completed |
| M12 | Observability | ✅ Completed |
| M13 | Argo CD | ✅ Completed |
| M14 | Prometheus | ✅ Completed (delivered under M12) |
| M15 | Grafana | ✅ Completed (delivered under M12) |
| M16 | Deployment Strategy (rolling update + K8s-native rollback) | ✅ Completed |
| M17 | Failure Testing | ✅ Completed |
| M18 | End-to-End DevSecOps (one continuous run) | ⬜ Not started |
| M19 | Documentation | ⬜ Not started (this document set addresses part of it) |

## How to Run

**Prerequisites:** Node.js 22+, Docker, Minikube, `kubectl`, Helm 3.

```bash
# 1. Clone both repositories
git clone https://github.com/trushang-dev/devops-poc-app.git
git clone https://github.com/trushang-dev/devops-poc-gitops.git

# 2. Run the services directly (no cluster needed)
cd devops-poc-app/services/user-service && npm install && npm start
# repeat for product-service (PORT 3002) and order-service (PORT 3003)

# — or — run all three with Docker Compose
cd devops-poc-app && docker compose up --build -d

# 3. Deploy to a local Kubernetes cluster
minikube start
minikube addons enable ingress
minikube addons enable metrics-server

cd devops-poc-gitops
helm upgrade --install devops-poc helm/microservices -n devops-poc --create-namespace

# 4. Hand deployment over to Argo CD (one manual step; everything after is Git-driven)
kubectl apply -n argocd -f argocd/application.yaml
```

**Verify it's healthy:**
```bash
kubectl get pods -n devops-poc
kubectl get hpa -n devops-poc
kubectl get application devops-poc -n argocd
curl http://$(minikube ip)/api/users/health
```

A more complete command-by-command checklist (Argo CD, Helm, Trivy, observability) is not duplicated here — ask if you'd like it published alongside this doc set.

## Key Learnings

- **CI ≠ CD.** GitHub Actions owns "produce a verified artifact" (test, scan, build, tag, push) plus one CD-*adjacent* step (updating the GitOps repo). Only Argo CD ever applies anything to Kubernetes.
- **Pull-based beats push-based for deployment security.** If CI held cluster credentials, any pipeline compromise (a bad dependency, a leaked secret, a malicious PR) becomes a direct path to production. Routing through Git plus a pull-based reconciler means CI's blast radius stops at "can write a Git commit."
- **Immutable, dual-tagged images pay off twice.** SemVer (`v1.0.0`) is what a human reasons about during an incident; the Git SHA answers "what exact source is this, unambiguously" regardless of release-naming decisions.
- **Scrape Pods, not Services, for per-replica metrics.** A `ServiceMonitor` against a multi-replica Service will have `kube-proxy` load-balance scrapes across replicas, corrupting counters that are kept in-memory per instance.
- **`.spec.replicas` needs one clear owner.** Both plain `helm upgrade` and Argo CD's `selfHeal` will fight an HPA over replica count unless explicitly told to ignore that field.
- **Readiness and liveness answer different questions.** Readiness controls traffic routing (fast, reversible, no restart); liveness controls container survival (slower, restarts the process). Testing them separately — not just configuring them — is the only way to see the actual difference.
- **Argo CD's reconciliation is spec-level, not Pod-level.** Pod deletion and in-container process signals are invisible to it; that's the ReplicaSet controller's and the kubelet's job respectively, not GitOps's.

## Known Limitations

- **PostgreSQL was scoped but never implemented.** The ConfigMap/Secret pairs per service (`DB_HOST`/`DB_PORT`/`DB_NAME`/`DB_USER`/`DB_PASSWORD`) are demo-only placeholders for a *future* database integration — no service has any database code or connection. All values are obviously fake (`.invalid` hostnames, `demo_...` usernames, `CHANGE-ME` passwords).
- **The OpenTelemetry Collector is deployed but currently idle.** Its `otlp` receiver has nothing sending to it, since the application uses `prom-client` (Prometheus format) rather than an OTel SDK. It's left in place as the front door for that, should it be added.
- **Grafana's admin password is a plaintext demo value** (`admin-demo-only-CHANGE-ME` in `helm/monitoring/kube-prometheus-stack-values.yaml`) — acceptable for a local POC, not representative of production secret management.
- **Argo CD's own installation isn't itself GitOps-tracked** — only the `Application` resource that Argo CD watches is stored in this repo; the `argocd` namespace's core components were applied by hand once.
- **The Argo CD UI doesn't fully work behind its path-based Ingress route** (`/argocd`): Argo CD v3.5.1's `server.basehref` doesn't patch the `<base href="/">` tag in its served `index.html`, so the UI's own JS/CSS bundle resolves to the wrong path in a browser (API/CLI access is unaffected). The standard nginx `sub_filter` workaround is blocked by this cluster's ingress-nginx admission webhook, which has snippet directives disabled. Left as a known, explicitly-deferred limitation rather than worked around.
- **Prometheus retention is a short 6-hour local window**, with no remote-write or long-term storage — appropriate for a learning cluster, not a system of record.
- **Minikube disk headroom is thin.** One session's cold start hit 94–96% disk usage and took ~7 minutes to settle (all pods briefly cycling through `CrashLoopBackOff` on probe timeouts) before resolving on its own. Not a code defect, but a standing environmental risk worth monitoring.
- **A Trivy Kubernetes/Helm config scan reports 9 HIGH findings**, all Pod-level `securityContext` hardening never explicitly set (read-only root filesystem, privilege escalation, dropped capabilities, seccomp profile) — reviewed and consciously deferred, not silently ignored.

## License

MIT — see [LICENSE](LICENSE).
