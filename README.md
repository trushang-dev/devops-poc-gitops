# DevOps Microservices POC — GitOps

The Kubernetes desired state for the [`devops-poc-app`](https://github.com/trushang-dev/devops-poc-app) microservices: Helm charts, Argo CD configuration, and an observability stack, all deployed to a local Kubernetes cluster with Git as the single source of truth.

![Architecture: CI/CD + GitOps + Kubernetes + Observability](docs/architecture.png)

## GitOps flow

```text
devops-poc-app (CI)
        │
        ▼
    Docker Hub
        │
        ▼
devops-poc-gitops  ◀── this repository
        │
        ▼
     Argo CD
        │
        ▼
   Kubernetes (Minikube)
```

Argo CD continuously compares the desired state declared in this Git repository against the actual state of the cluster and reconciles any difference — including reverting manual, out-of-band `kubectl` changes (`selfHeal`) and removing resources no longer defined in Git (`prune`).

## What's deployed

- **3 microservices** (user, product, order) — 2 replicas each, `HorizontalPodAutoscaler` on CPU, readiness/liveness probes, resource requests/limits, RollingUpdate
- **NGINX Ingress** — path-based routing for the application API, Argo CD UI, and the monitoring stack behind a single external IP
- **ConfigMap + Secret per service** — demo configuration for a future database integration (see [Configuration note](#configuration-note) below)
- **Argo CD** — the `devops-poc` Application, self-healing and pruning, syncing `helm/microservices` from `main`
- **Observability** — Prometheus, Grafana, and an OpenTelemetry Collector scraping/visualizing each service's `/metrics` endpoint and HTTP latency

## Repository structure

```text
devops-poc-gitops/
├── kubernetes/          # raw manifests (M4–M7 historical reference)
│   ├── user-service/  product-service/  order-service/
│   └── monitoring/
├── helm/
│   ├── microservices/   # Chart.yaml, values.yaml, templates/ — the live-managed release
│   └── monitoring/      # kube-prometheus-stack + OpenTelemetry Collector values
├── argocd/
│   ├── application.yaml # the Argo CD Application watching this repo
│   └── ingress.yaml
└── docs/                 # milestone-by-milestone build notes (M0–M15)
```

Helm (release name `devops-poc`) manages the live cluster resources; the raw `kubernetes/` manifests are kept as a historical reference from before Helm/Argo CD were introduced.

## Configuration note

Each service ships a demo `ConfigMap`/`Secret` (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`) representing configuration for a **future** database integration the application does not yet connect to. All values are explicitly fake (`.invalid` hostnames, `demo_...` usernames, `CHANGE-ME` passwords) and every manifest says so in a header comment. This exists purely to demonstrate Kubernetes configuration management — a Kubernetes `Secret` is base64-encoded, not encrypted, and should never hold real production credentials.

## Applying this to a cluster

```bash
# Argo CD Application — the one manual step; everything after this is Git-driven
kubectl apply -n argocd -f argocd/application.yaml

# or, without Argo CD, drive the Helm release directly
helm upgrade --install devops-poc helm/microservices -n devops-poc --create-namespace
```

Once the Argo CD `Application` is applied, do not run `helm upgrade` against this release manually — Argo CD owns reconciliation and will fight (or be fought by) manual changes.

## Documentation

Detailed, milestone-by-milestone build notes — what was built, why, and what was verified live — are in [`docs/`](docs/), covering Git strategy, containerization, CI, Kubernetes fundamentals, Ingress, ConfigMaps/Secrets, HPA, Helm, GitOps, Docker Hub, observability, and Argo CD.

## Rules this repo follows

- Git is the single source of truth for cluster state.
- No real secrets are ever committed (see [Configuration note](#configuration-note)).
- Images are deployed by versioned tag, never `latest`.
- No hardcoded pod IPs.
- Argo CD — not `kubectl` or CI — owns deployment to the cluster.

## License

MIT — see [LICENSE](LICENSE).
