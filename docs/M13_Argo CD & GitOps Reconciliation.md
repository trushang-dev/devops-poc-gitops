# M13 — Argo CD & GitOps Reconciliation

### What is it used for

* Automatically deploy Kubernetes changes from the **GitOps repository**.
* Continuously compare **desired state in Git** with **actual state in Kubernetes**.
* Detect changes and reconcile Kubernetes automatically.
* Provide deployment visibility through the Argo CD GUI.

### Key points to review / make strong

* Argo CD
* GitOps reconciliation
* Desired state vs actual state
* `Application` resource
* `source`
* `targetRevision`
* `path`
* `destination`
* Automated sync
* `selfHeal`
* `prune`
* `Synced` vs `OutOfSync`
* `Healthy` vs `Degraded`
* Drift detection
* Rollback through Git
* Why Argo CD owns Kubernetes deployment
* Why GitHub Actions must **not** directly deploy to Kubernetes

### How it is implemented in our POC

We created an Argo CD Application:

```text
devops-poc
```

It watches:

```text
GitHub
└── devops-poc-gitops
    └── main
        └── helm/microservices
```

and deploys to:

```text
Minikube
└── devops-poc namespace
```

The overall flow is:

```text
GitOps Repository
       │
       │ watch
       ▼
    Argo CD
       │
       │ reconcile
       ▼
   Kubernetes
       │
       ▼
 Application Pods
```

### Core logic

Argo CD continuously asks:

> "Does Kubernetes currently match what Git says should exist?"

Example:

Git says:

```yaml
image:
  tag: "9858175..."
```

Kubernetes is running:

```text
9858175...
```

Therefore:

```text
Desired State = Actual State
              ↓
            Synced
```

Now suppose Git changes:

```yaml
image:
  tag: "abc123..."
```

Argo CD detects:

```text
Desired State ≠ Actual State
```

and with automated sync:

```text
Git change
   ↓
Argo CD detects
   ↓
Helm renders manifests
   ↓
Kubernetes updated
   ↓
Rolling deployment
   ↓
New Pods
```

### Important Argo CD configuration

Our Application uses:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

#### `automated`

Argo CD automatically synchronizes changes instead of requiring someone to click **Sync**.

#### `selfHeal`

If someone manually changes a managed Kubernetes resource:

```text
Git = desired state
K8s = manually modified
```

Argo CD can detect the drift and restore the Git-defined state.

#### `prune`

If a resource is removed from the Git-defined configuration, Argo CD can remove the corresponding Kubernetes resource.

So remember:

```text
selfHeal → fix drift

prune → remove resources no longer declared in Git
```

### Important concept — Argo CD Application

The `Application` is essentially the instruction telling Argo CD:

```text
WHERE is my Git repository?
        ↓
WHICH branch?
        ↓
WHICH path?
        ↓
WHERE should I deploy?
```

Our POC:

```text
Repository:
devops-poc-gitops

Branch:
main

Path:
helm/microservices

Cluster:
https://kubernetes.default.svc

Namespace:
devops-poc
```

### Argo CD GUI

Our POC exposes the Argo CD UI through:

```text
http://192.168.49.2/argocd/
```

The important things to review in the UI are:

```text
Application
   │
   ├── Sync Status
   │      ├── Synced
   │      └── OutOfSync
   │
   ├── Health
   │      ├── Healthy
   │      └── Degraded
   │
   └── Resources
          ├── Deployment
          ├── ReplicaSet
          ├── Pods
          └── Services
```

### Important factor to remember

**Argo CD is not your CI system.**

It does not primarily:

```text
❌ run tests
❌ build Docker images
❌ scan source code
❌ build application artifacts
```

That's CI's job.

Argo CD's job is:

```text
Git desired state
       ↓
Reconcile
       ↓
Kubernetes actual state
```

### Our complete CI → CD flow

Now the previous milestones connect:

```text
                 APPLICATION REPO
                       │
                       ▼
                GitHub Actions
                       │
             ┌─────────┴─────────┐
             │                   │
          Test/Scan          Docker Build
                                 │
                                 ▼
                             Docker Hub
                                 │
                                 ▼
                         GitOps Repository
                         image.tag = SHA
                                 │
                                 ▼
                              Argo CD
                                 │
                         Reconciliation
                                 │
                                 ▼
                           Kubernetes
                                 │
                                 ▼
                              Pods
```

**Mental shortcut:**

> **GitHub Actions builds it.
> Docker Hub stores it.
> GitOps declares it.
> Argo CD deploys/reconciles it.
> Kubernetes runs it.**
