# Project Journey: M0 → M17

This is the build story behind the DevOps/GitOps POC — what each phase set out to do, what actually went wrong along the way, how it was verified, and what it taught. For the current-state reference, see [README.md](README.md); for the architectural reasoning, see [ARCHITECTURE.md](ARCHITECTURE.md).

Eighteen milestones are grouped into eight phases below, in the order they were built. Every claim of "completed" here is backed by a specific verification step — most of them live command output or controller-log timestamps captured in this project's internal completion notes for M11, M13, M16, M17, and M18 (kept outside this public repository). Nothing below is inferred from configuration alone unless stated as such.

## Phase 1 — Foundation & Application (M0–M2)

**Goal:** Establish two separate Git repositories — one for application code, one for deployment/GitOps state — and get three independently runnable Node.js services working, before any DevOps automation existed.

**Implementation:** `devops-poc-app` (three Express services: `user-service`, `product-service`, `order-service`, each with an in-memory dataset and a `/health` endpoint) and `devops-poc-gitops` (empty at this point, reserved for Kubernetes/Helm/Argo CD state) were created as two independent repositories. A `feature/* → develop → release/* → main` branching model was adopted for both.

**Challenge:** None technical — the real decision here was architectural: committing early to keeping application code and deployment state in physically separate repositories, before there was any concrete reason yet to need that separation. That decision is what made the CI/CD-vs-GitOps split possible later; retrofitting it after the fact would have been much harder.

**Verification:** Both repositories exist independently on GitHub with the expected branch structure (`git branch -a`, `git remote -v`); each service runs and responds on its assigned port with `npm start`.

**Learning:** GitOps depends on the application repo and the "desired state" repo being genuinely separate — not a folder convention inside one repo, but two independently-permissioned, independently-versioned Git histories. `devops-poc-app` produces artifacts; `devops-poc-gitops` declares what should run. That split is the foundation everything else in this project sits on.

## Phase 2 — Containerization & Local Kubernetes (M3–M5)

**Goal:** Package each service into a Docker image, run all three on a local Minikube cluster with Deployments/Services, and expose them through a single Ingress.

**Implementation:** Each service got its own single-stage `Dockerfile` (`node:22-alpine`, non-root `node` user, `npm ci --only=production`). `devops-poc-gitops/kubernetes/` gained a Namespace, one Deployment + `ClusterIP` Service per service (readiness/liveness probes on `/health`, resource requests/limits, `RollingUpdate` strategy), and later an NGINX Ingress routing `/api/{users,products,orders}` (and their `/health` sub-paths) to the matching Service.

**Challenge:** `ingress-nginx` evaluates regex-based path rules in the order they appear in the manifest, not by specificity. The general `/api/users(...)` rule was initially matching health-check requests meant for the more specific `/api/users/health` rule. Fixed with a negative lookahead so the two rules became mutually exclusive (commit `8cbad48`).

**Verification:** All three services reachable via `http://$(minikube ip)/api/{users,products,orders}` and their `/health` variants (7 routes total, including one deliberately invalid path returning 404); a full scale-up/rolling-update/rollback cycle was exercised manually at this stage (2→3→2 replicas, a new revision via an added env var with zero pods lost, then a deliberately bad image tag → `ImagePullBackOff` → `kubectl rollout undo` → confirmed healthy again).

**Learning:** Ingress rule ordering is a real, easy-to-miss footgun with regex paths — "more specific" doesn't automatically win; file order does, unless the rules are written to be mutually exclusive.

## Phase 3 — Configuration, Reliability & Packaging (M6–M8)

**Goal:** Separate configuration from images (ConfigMap/Secret), add autoscaling on top of the existing reliability primitives, and package everything as a Helm chart instead of hand-duplicated YAML.

**Implementation:** One ConfigMap (`DB_HOST`/`DB_PORT`/`DB_NAME`) and one Secret (`DB_USER`/`DB_PASSWORD`) per service, consumed via `env[].valueFrom`, representing configuration for a **future** database the application never actually connects to — a deliberate, clearly-labeled demonstration of Kubernetes configuration management, not a real integration. An HPA (`autoscaling/v2`, `minReplicas: 2`, `maxReplicas: 3`, 70% CPU target) was added per service on top of the probes/resources/RollingUpdate strategy that already existed from M4. All of it was then repackaged as a single Helm chart (`helm/microservices`) templating over a `services` list in `values.yaml`, replacing hand-maintained per-service YAML.

**Challenge:** Two real issues surfaced while cutting over to Helm. First, `helm --set` on a list index (`services[0].replicas=3`) silently drops the other keys on that list item — caught by a template render failing before it ever touched the cluster. Second, once the HPA takes live ownership of `.spec.replicas` via the scale subresource, a plain `helm upgrade`'s server-side apply can conflict with it; resolved with `--reset-values` to reconverge Helm's view with the HPA's. Also, a ConfigMap edit was shown to **not** propagate to already-running pods until an explicit `kubectl rollout restart` — Kubernetes has no built-in env-var hot-reload.

**Verification:** `helm template` output produced **zero diff** (`kubectl diff`) against the still-running raw-manifest resources before cutover; a clean values-driven change (memory limit 128Mi→256Mi) was applied, confirmed via `helm get values`, then reverted with `helm rollback` and confirmed restored.

**Learning:** A Kubernetes Secret's real value here isn't encryption (it's base64, not encrypted) — it's the type-level separation and default-hidden `describe` output. And once an HPA and a templating tool can both write to the same field, one of them needs an explicit exception, or they will fight each other indefinitely.

## Phase 4 — Security & CI Automation (M9–M11)

**Goal:** Establish a Trivy scanning policy across filesystem, image, and Kubernetes/Helm config layers, then automate the entire test → scan → build → publish → GitOps-update chain in GitHub Actions.

**Implementation:** Trivy scans were run manually first (M9) to establish a severity policy — CRITICAL blocks, HIGH is reviewed and documented (not auto-blocking), MEDIUM/LOW are logged only — before any CI existed to enforce it. `ci.yml` (M10) then wired `npm ci → npm test → Trivy fs scan → Docker build → Trivy image scan → Docker Hub push`, with images published under `trushangdev/devops-poc-<service>` (M11).

**Challenge:** The pipeline's first working version tagged images by Git SHA only and published on **every** `develop` push — functional, but it meant `develop` (an integration branch, not a release) was producing deployable artifacts. This was corrected (`fix(ci): gate Docker release on release branches`, commit `7627a90`) to the current model: `develop`/PR runs are validation-only, and only a `release/*` branch push builds, scans, publishes, and updates GitOps — with SemVer as the primary deployed tag and the Git SHA kept as a secondary traceability tag. Separately, the Helm deployment template initially rendered `devops-poc/<service>` as the image path, which didn't match any real Docker Hub repository; fixed (commit `d13b3e4`) to render `trushangdev/devops-poc-<service>`. One CRITICAL Trivy finding (`CVE-2026-59873`, in `tar`, bundled with npm inside the `node:22-alpine` base image) was investigated and confirmed unreachable from the application's own dependency tree (0 vulnerabilities in `package-lock.json` for all three services) — documented as an explicit, reviewed `.trivyignore` exception rather than a blanket suppression.

**Verification:** A real pull request into `develop` triggered a `pull_request` CI run; the merge triggered a `push` run whose success is evidenced by the GitOps commit it produced. A real `release/v1.0.0` push was later exercised end-to-end: images confirmed running in Kubernetes at the expected `v1.0.0` tag, reconciled automatically by Argo CD, with zero manual deployment steps (exact commit hashes and timestamps were captured in the project's internal M11 completion notes; some artifacts — the raw GitHub Actions run log, a direct Docker Hub query — could not be independently observed in that session due to the lack of a `gh` CLI/API token, and are honestly reported as inferred-from-outcome rather than directly confirmed).

**Learning:** "It's on a validation branch" and "it's on a release branch" need to be different pipelines, not the same pipeline running unconditionally. And immutable, dual-tagged images (SemVer for humans, SHA for exact traceability) cost nothing extra to produce once CI exists, so there's little reason not to keep both.

## Phase 5 — Observability (M12, M14–M15)

**Goal:** Deploy Prometheus, Grafana, and an OpenTelemetry Collector, instrument the application with real metrics, and build a dashboard that actually reflects application and cluster health.

**Implementation:** `kube-prometheus-stack` (Prometheus + Grafana + `kube-state-metrics` + Node Exporter + Prometheus Operator) plus a separate OpenTelemetry Collector chart, both in a dedicated `devops-monitoring` namespace. Each service was instrumented with `prom-client` — an `http_requests_total` counter and `http_request_duration_seconds` histogram, both labeled `method`/`route`/`status_code` — exposed on `/metrics`. A "DevOps POC Overview" Grafana dashboard was provisioned as code via a ConfigMap and the chart's sidecar — no manual dashboard import. (A later, separate change added a fifth and sixth row — pod restarts/readiness and Node.js runtime internals via `prom-client`'s default metrics — bringing the dashboard to 18 panels across six rows; not part of M12 itself, but built on the same PodMonitor-based pipeline M12 established.)

**Challenge:** Two real, independently-discovered bugs. First, scraping the application through a `ServiceMonitor` (i.e., the Service's `ClusterIP`) let `kube-proxy` load-balance each scrape across both replicas; since each replica keeps its own independent in-memory counter, this produced a "counter" that jumped between two unrelated values instead of increasing monotonically — confirmed directly (two `order-service` pods separately reported 35 and 31 requests, with no way to reconcile them from the Service alone). Fixed by scraping each Pod individually via a `PodMonitor`. Second, the OpenTelemetry Collector's own `prometheus` receiver was independently scraping those same `/metrics` endpoints, producing a second, duplicate metric series under a different `job` label that badly distorted rate calculations (confirmed ~120× the correct rate during a later audit). Fixed by removing that receiver configuration entirely — the Collector now only exposes its own self-telemetry and stands ready (via its still-configured, currently-unused `otlp` receiver) for future OTel SDK instrumentation that doesn't exist yet.

**Verification:** All PodMonitor/ServiceMonitor targets confirmed `up`; a real `ab -n 10000 -c 20` run against `user-service` completed with 0 failed requests, the counter increasing by exactly 10,000, real p95 latency recorded, and the HPA correctly scaling only `user-service` (2→3) while the other two services stayed at 2/2.

**Learning:** "Metrics exist" and "metrics are correct" are different claims — a load test with a known expected delta (10,000 requests) is what actually proves a counter isn't lying, not just confirming a Prometheus target shows `up`.

## Phase 6 — GitOps Reconciliation with Argo CD (M13)

**Goal:** Put Argo CD in charge of reconciling `devops-poc-gitops@main` onto the cluster, and prove — not just configure — drift detection, self-healing, and Git-based rollback.

**Implementation:** A single Argo CD `Application` (`devops-poc`, namespace `argocd`) watching `devops-poc-gitops@main` at path `helm/microservices`, with `syncPolicy.automated: {prune: true, selfHeal: true}` and an explicit `ignoreDifferences` on `Deployment.spec.replicas` so the HPA (M7) and Argo CD's reconciliation don't fight each other.

**Challenge:** Proving self-healing and rollback required more than trusting the `selfHeal: true` flag — both the drift-and-revert cycle and the rollback deploy/revert cycle completed in about 4 seconds, faster than live 2-second polling could reliably catch. The controller's own timestamped logs (`"Updated sync status: Synced -> OutOfSync"` / `"-> Synced"`) had to be pulled directly to produce real evidence, rather than inferring success from the end state alone.

**Verification:** A live, out-of-band `kubectl set env` change was detected and automatically reverted within 4 seconds, independently corroborated by a new `ReplicaSet` appearing and being scaled back to zero. Separately, a real state change (`v1.0.0` → a second proven-good SHA-tagged image) was promoted through a normal feature-branch/PR merge, auto-deployed by Argo CD, then reverted with `git revert` (not `reset`, not a manual `kubectl` command) on a second PR — Argo CD auto-restored the original state within 4 seconds of that merge too. All 7 of the demonstration items originally scoped for Argo CD (Application creation, Git sync, health status, `OutOfSync` detection, drift detection, self-heal, Git-based rollback) now have direct evidence behind them.

**Learning:** A sync policy flag being set is a configuration fact, not a behavioral proof. Watching the actual detection-to-correction cycle happen — via logs, not inference — is what turns "GitOps should self-heal" into "GitOps was observed to self-heal in N seconds, here's the log."

## Phase 7 — Deployment Strategy & Rollback (M16)

**Goal:** Demonstrate the Kubernetes-native rollback mechanism (`kubectl rollout undo`) as a genuinely distinct capability from the GitOps-native rollback already proven in M13 — not the same thing described twice.

**Implementation:** No new configuration was needed — the `RollingUpdate` strategy (`maxSurge: 1`, `maxUnavailable: 1`) had existed since M8's Helm packaging. The milestone's actual deliverable was the live demonstration itself: a rolling update from `v1.0.0` to an already-proven-good image, followed by `kubectl rollout undo` — with availability sampled every 2 seconds throughout to prove the rollout never dropped below one ready pod.

**Challenge:** `selfHeal: true` would revert any `kubectl`-driven image change within seconds, exactly as measured in M13's drift test — which would have made a clean Kubernetes-native rollback demo impossible without interference. Argo CD's automated sync was **temporarily** paused (`kubectl patch application devops-poc ... syncPolicy.automated=null`) for the duration of the demo only, and restored to the exact original policy immediately after — confirmed via `kubectl get application ... -o jsonpath` before, during, and after. Separately, an unrelated environmental issue (a rocky Minikube cold start at 94–96% disk usage, all pods briefly `CrashLoopBackOff` on probe timeouts) had to settle on its own (~7 minutes) before the demo could safely begin — disclosed as a standing risk, not hidden.

**Verification:** New revisions 13 (the update) and 14 (the undo) appeared in `kubectl rollout history`; pod images were directly inspected before and after each transition; once Argo CD's sync was restored, it reported `Synced`/`Healthy` with **no drift to reconcile** — direct proof the Kubernetes-native rollback had landed the cluster exactly where Git already said it should be.

**Learning:** These two rollback paths solve different problems. `kubectl rollout undo` is the fast, local, in-cluster undo you reach for while actively debugging right now — it leaves no Git record and doesn't need Argo CD at all. `git revert` is the durable, reviewable, "this is now our desired state" record — and it's the only one of the two Argo CD will keep enforcing after you walk away.

## Phase 8 — Failure Testing (M17)

**Goal:** Deliberately induce the four failure scenarios scoped in `PROJECT_SCOPE.md` §19, and — the actual point of this milestone — keep pod recreation, readiness behavior, liveness/restart behavior, and GitOps self-healing as four clearly distinct mechanisms rather than one blurred "Kubernetes heals itself" story.

**Implementation:** Two of the four scenarios (bad image/rollback, GitOps drift) already had direct evidence from M4/M16 and M13 respectively and were deliberately not re-run. The two new tests: deleting a running pod directly, and freezing the real Node process inside a pod (`kill -STOP` on PID 19 — confirmed via `ps aux` that PID 1 is only the `npm start` wrapper) to produce one honest, sequential readiness-then-liveness failure, rather than inventing a fake failure code path that doesn't exist in the application.

**Challenge:** The application's `/health` handler is an unconditional 200 responder with no failure branch — there was no way to make it "honestly" fail through the API. The chosen approach (freezing the actual process) produces a real, unscripted failure using the exact same detection path a genuine hang would trigger, rather than adding test-only code to the application to simulate one.

**Verification:** Pod deletion → a replacement was created by the ReplicaSet controller (not Argo CD) within ~16 seconds (`SuccessfulCreate` event). Process freeze → readiness failed at ~32 seconds (pod marked `Ready: false` *and* simultaneously removed from the Service's endpoint list — direct proof of a routing change, not just a status flag), then liveness triggered a real container restart at ~101 seconds (restart count 0→1, confirmed via `Killing`/`Unhealthy` events and fresh `Created`/`Started` events), with full automatic recovery and zero manual intervention. Neither test required pausing Argo CD — both are invisible to its spec-level diffing, and `devops-poc` stayed `Synced` throughout.

**Learning:** "Self-healing" is doing a lot of work as a single word in most descriptions of Kubernetes. This project alone required four separate mechanisms to cover four separate failure classes — a ReplicaSet controller, a kubelet-driven restart, a Service endpoint update, and a GitOps reconciler — and conflating any two of them into "the same thing" would misdescribe how the recovery actually happened.

## Phase 9 — End-to-End DevSecOps (M18)

**Goal:** Every individual pipeline capability had been proven by this point — but each was proven in its own milestone, often against a hand-picked, low-risk change (M13's rollback test deliberately used an already-proven-good image specifically to avoid conflating GitOps mechanics with image risk). M18 closes that gap: exercise `PROJECT_SCOPE.md` §24's full success-criteria chain (items 1–15) as one continuous, real delivery — a genuine code change traced end to end — rather than fifteen separate, independently-asserted claims.

**Implementation:** The smallest possible end-user-observable change was used: `user-service`'s `/version` endpoint, `1.0.0` → `1.0.1` (the only such endpoint across the three services). The change went through the full, real path: `feature/version-1.0.1-bump` → PR #5 → `develop` → `release/v1.0.1` → the existing, unmodified CI pipeline → Docker Hub → an automated GitOps commit (`ad8d207`, picked up by Argo CD directly, not merged by hand) → a Kubernetes rollout across all three services (all six pods land on `v1.0.1`, since `values.yaml` uses one shared `image.tag` for all three).

**Challenge:** Proving the code — not just the tag — had actually changed required going past `kubectl get pods -o jsonpath` (which only proves what image *string* is set). The defining piece of evidence was `kubectl exec deploy/user-service -- wget -qO- http://localhost:3001/version` returning `{"service":"user-service","version":"1.0.1"}` — read directly from inside the live container. Separately, this run surfaced a genuine, pre-existing bug: `GET /api/users/version` through the Ingress 404s (`Route GET /users// does not exist`) — a double-slash artifact of the existing regex `rewrite-target` rule, specific to the `/version` path shape (the equivalent `/health` routes work correctly). Not fixed, since it's unrelated to the CI/CD chain itself — worked around by verifying via `kubectl exec` instead, which is arguably stronger evidence anyway since it bypasses the Ingress layer entirely.

**Verification:** Total elapsed time from the `release/v1.0.1` push to all six pods running `v1.0.1` was under 6 minutes, entirely unattended. `up{job=~"devops-poc.*"}` confirmed all six new pods scraping healthy in Prometheus, with real recorded traffic (`sum(http_requests_total{job="devops-poc/user-service"})` returning a real, non-zero count). Argo CD ended `Synced`/`Healthy` at the new revision. Full detail, exact timestamps, and the complete stage-by-stage evidence table were captured in the project's internal M18 completion notes (kept outside this public repository).

**Learning:** An image tag changing is not proof that new code is running — a tag is just a label, and labels can be wrong (mistagged builds, a stale cache, a manual `kubectl set image` typo). The only conclusive proof is reading the behavior back from inside the running container itself. Separately: chaining every previously-proven capability into one real, continuous run surfaces integration gaps (like the shared-tag-across-services behavior, and the `/version` Ingress bug) that no individual milestone's narrower scope would ever have exposed.

## What's Next

Per `PROJECT_SCOPE.md` §20, one milestone remains **not started**:

- **M19 — Documentation.** This document set (`README.md`, `ARCHITECTURE.md`, `PROJECT_JOURNEY.md`, and `docs/RUNBOOK.md`, `docs/TROUBLESHOOTING.md`, `docs/OBSERVABILITY.md`, `docs/SECURITY.md`) covers the substance of what M19 asks for. What's left is mostly review and polish rather than new content: keeping the doc set consistent as the project evolves (dashboard changes, new milestones), and optionally consolidating `DEVOPS_POC_LOCAL_REVIEW_CHECKLIST.md`'s command reference into `docs/RUNBOOK.md` if a single canonical checklist is wanted.
