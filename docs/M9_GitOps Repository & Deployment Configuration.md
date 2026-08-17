# M9 — GitOps Repository & Deployment Configuration

### What is it used for

* Store the **desired Kubernetes state** separately from application source code.
* Make Git the **source of truth** for what should run in Kubernetes.
* Prepare the project for automated deployment through **Argo CD**.

### Key points to review / make strong

* GitOps principle
* Desired state vs actual state
* Application repo vs GitOps repo
* Kubernetes manifests vs Helm chart
* Immutable image tags
* Why deployment configuration belongs in Git
* Separation of CI and CD
* Pull-based deployment model

### How it is implemented in our POC

We have two repositories:

```text
devops-poc-app
        │
        │ application source
        ▼
   GitHub Actions
        │
        ▼
    Docker Hub
```

and:

```text
devops-poc-gitops
        │
        │ desired Kubernetes state
        ▼
      Argo CD
        │
        ▼
    Kubernetes
```

The GitOps repository contains:

```text
devops-poc-gitops
│
├── helm/
│   ├── microservices/
│   └── monitoring/
│
├── kubernetes/
│   └── monitoring/
│
└── argocd/
```

### Core logic

The key idea is:

```text
Application Code
      ↓
Docker Image
      ↓
GitOps Repository
      ↓
Argo CD
      ↓
Kubernetes
```

For a new application version:

```text
Developer changes code
        ↓
GitHub Actions
        ↓
Build + Test + Scan
        ↓
Push Docker image
        ↓
Update image SHA in GitOps repo
        ↓
GitOps repo changes
        ↓
Argo CD detects change
        ↓
Helm renders desired state
        ↓
Kubernetes updated
```

### Desired state vs actual state

This is the **most important GitOps concept** to understand.

Git says:

```text
user-service image = version B
replicas = 2
```

Kubernetes currently has:

```text
user-service image = version A
replicas = 2
```

Therefore:

```text
Desired State ≠ Actual State
```

Argo CD detects this and reconciles Kubernetes toward the Git-defined state.

Once corrected:

```text
Desired State = Actual State
```

### Important factor to remember

**GitOps is not simply "put YAML in Git."**

The important part is the reconciliation model:

```text
Git
 ↓
Desired State
 ↓
Argo CD continuously watches
 ↓
Kubernetes Actual State
```

Our POC follows a **pull-based deployment model**:

> GitHub Actions does not directly deploy to Kubernetes. Argo CD reads the GitOps repository and performs reconciliation.

That separation is the foundation for **M10/M11 CI automation and M13 Argo CD**.
