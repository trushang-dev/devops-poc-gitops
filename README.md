# DevOps POC — GitOps

This repository contains the Kubernetes desired state for the local DevOps Microservices POC.

## Responsibility

This repository contains:

- Kubernetes configuration
- Helm charts
- Environment values
- Argo CD configuration
- Application image versions
- GitOps deployment state

Application source code belongs to `devops-poc-app`.

## GitOps Flow

```text
Application Repository
        ↓
GitHub Actions
        ↓
Docker Hub
        ↓
GitOps Repository
        ↓
Argo CD
        ↓
Local Kubernetes
```

Git is the source of truth for the desired Kubernetes state.

Argo CD compares:

```text
Git Desired State
       vs
Kubernetes Actual State
```

and reconciles differences.

## Target Structure

```text
devops-poc-gitops/
├── helm/
├── environments/
├── argocd/
└── README.md
```

The directories will be created progressively according to the project milestones.

## Kubernetes

The final local environment will use `Minikube` and will manage:

- User Service
- Product Service
- Order Service
- PostgreSQL
- NGINX Ingress
- ConfigMaps
- Secrets
- Persistent storage
- Health checks
- Scaling

## Helm

Helm will eventually package the application Kubernetes resources.

Container image versions will be managed through Helm values.

Example:

```yaml
userService:
  image:
    repository: docker.io/<username>/user-service
    tag: v1.0.0
```

## Argo CD

Argo CD will:

- Watch the GitOps repository
- Detect changes
- Synchronize Kubernetes resources
- Report application health
- Detect configuration drift

## Important Rules

- Git is the source of truth.
- Do not commit real secrets.
- Use versioned image tags.
- Avoid `latest` for deployment versions.
- Do not use hardcoded pod IPs.
- Do not bypass Argo CD for normal deployments.
- Do not directly deploy with `kubectl` from CI.
- Follow `../PROJECT_SCOPE.md`.
- Follow `../CLAUDE OPERATING RULES.md`.

## Deployment Example

```text
Application release:
v1.2.0

Docker image:
user-service:v1.2.0

GitOps change:
tag: v1.2.0

Argo CD:
sync

Kubernetes:
user-service v1.2.0
```

## Repository Relationship

```text
devops-poc-app
      ↓
GitHub Actions
      ↓
Docker Hub
      ↓
devops-poc-gitops
      ↓
Argo CD
      ↓
Kubernetes
```