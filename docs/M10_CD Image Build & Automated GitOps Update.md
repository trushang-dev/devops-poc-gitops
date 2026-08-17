# M10 — CI/CD Image Build & Automated GitOps Update

### What is it used for

* Connect **application CI** with the **GitOps deployment flow**.
* Automatically build and publish a new Docker image after code changes.
* Automatically update the GitOps repository with the **new image SHA**.
* Remove the manual step of changing the image tag.

### Key points to review / make strong

* CI vs CD separation
* GitHub Actions job dependencies
* Docker image tagging with Git SHA
* Docker Hub authentication using GitHub Secrets
* Trivy filesystem scan
* Trivy image scan
* `needs:` between jobs
* GitOps repository update
* Git commit + push from CI
* Why the image tag should be immutable
* Why GitHub Actions does **not** deploy directly to Kubernetes

### How it is implemented in our POC

The flow is:

```text
Developer
    ↓
Push / Merge to develop
    ↓
GitHub Actions
    ↓
┌─────────────────────────────┐
│ Test                        │
│ Trivy filesystem scan       │
│ Docker build                │
│ Trivy image scan            │
│ Docker Hub push             │
└─────────────────────────────┘
    ↓
Update GitOps
    ↓
devops-poc-gitops/main
    ↓
image.tag = Git SHA
```

For example:

```text
Application commit:
9858175a...
        ↓
Docker image:
trushangdev/devops-poc-user-service:9858175a...
        ↓
GitOps values.yaml:
image:
  repository: trushangdev
  tag: "9858175a..."
```

### Core logic

There are effectively **two automation stages**.

#### Stage 1 — Build the application artifact

```text
Code
 ↓
Test
 ↓
Security Scan
 ↓
Docker Build
 ↓
Image Scan
 ↓
Docker Hub
```

Only after the required checks succeed does the pipeline proceed.

#### Stage 2 — Update deployment state

The CI job updates:

```yaml
image:
  repository: trushangdev
  tag: "<new-git-sha>"
```

in the GitOps repository.

Then:

```text
GitOps commit
     ↓
Argo CD detects change
     ↓
Helm chart renders new image
     ↓
Kubernetes rollout
```

### Important factor to remember

This is the **most important M10 concept**:

> **CI produces the artifact; Git becomes the deployment trigger.**

GitHub Actions:

```text
Build + Test + Scan + Push Image
              ↓
       Update GitOps Git
```

Argo CD:

```text
GitOps Git
    ↓
Detect change
    ↓
Reconcile
    ↓
Kubernetes
```

So **GitHub Actions and Argo CD have different responsibilities**.

| Component      | Responsibility                   |
| -------------- | -------------------------------- |
| GitHub Actions | Build, test, scan, publish image |
| Docker Hub     | Store image                      |
| GitOps repo    | Store desired deployment version |
| Argo CD        | Reconcile Git → Kubernetes       |
| Kubernetes     | Run the application              |

### One critical detail from our POC

We use **Git SHA rather than `latest`**.

Bad:

```text
user-service:latest
```

Better:

```text
user-service:9858175a...
```

Because the SHA gives us:

* Exact version identification
* Reproducibility
* Traceability
* Easier rollback
* No ambiguity about what Kubernetes should run

### Mental model

Remember M10 as:

```text
CODE
 ↓
CI
 ↓
TEST + SCAN
 ↓
DOCKER IMAGE
 ↓
DOCKER HUB
 ↓
GITOPS UPDATE
 ↓
ARGO CD
 ↓
KUBERNETES
```

**M10 is where our POC stops being mostly manual and starts becoming a real automated GitOps delivery pipeline.**
