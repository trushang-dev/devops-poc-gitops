# M3 — GitHub Actions CI Pipeline

### What is it used for

* Automatically **validate application code** whenever changes are pushed.
* Remove manual testing/build steps.
* Create a repeatable CI process before an application becomes a deployable artifact.

### Key points to review / make strong

* GitHub Actions workflow
* Workflow → Job → Step
* Triggers: `push` / `pull_request`
* Runners
* GitHub Actions secrets
* Dependency installation
* Automated tests
* Build process
* Job dependencies using `needs`
* CI vs CD
* Why CI should fail early

### How it is implemented in our POC

Our CI lives in the **application repository**:

```text
devops-poc-app
└── .github/
    └── workflows/
        └── ci.yml
```

The workflow handles the application-side automation.

Conceptually:

```text
Developer
    ↓
git push / Pull Request
    ↓
GitHub Actions
    ↓
Install dependencies
    ↓
Run tests
    ↓
Security scan
    ↓
Docker build
    ↓
Docker image
```

Later stages are connected to the GitOps flow.

### Core logic

The important concept is that **CI validates and builds**.

```text
Source Code
     ↓
GitHub Actions
     │
     ├── Test
     │
     ├── Trivy filesystem scan
     │
     ├── Docker build
     │
     ├── Trivy image scan
     │
     └── Push image
              ↓
          Docker Hub
```

Our POC uses the **Git commit SHA as the Docker image tag**, giving us traceability between source code and the container artifact.

Example:

```text
Git commit
9858175...
     ↓
Docker image
trushangdev/devops-poc-user-service:9858175...
```

### Important factor to remember

The biggest distinction:

> **CI does not mean deployment.**

In our POC:

```text
GitHub Actions
      ↓
Test + Scan + Build + Push
      ↓
Docker Hub
```

It does **not** do:

```text
GitHub Actions ──X──→ Kubernetes
```

Kubernetes deployment is handled later through the **GitOps + Argo CD** flow.

So remember M3 as:

> **Code enters GitHub Actions → code is tested and packaged into a trusted Docker artifact.**
