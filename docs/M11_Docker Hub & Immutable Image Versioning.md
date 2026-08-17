# M11 — Docker Hub & Immutable Image Versioning

### What is it used for

* Store the Docker images produced by CI in a **container registry**.
* Make application images available for Kubernetes to pull.
* Use an immutable **Git SHA tag** to identify exactly which source version produced each image.

### Key points to review / make strong

* Docker Registry vs Docker Image
* Docker Hub authentication
* Repository naming
* Image tags
* Immutable tags
* Git SHA → Docker image relationship
* `imagePullPolicy`
* Why `latest` is avoided
* How Kubernetes pulls an image
* How GitOps references the image

### How it is implemented in our POC

Our Docker images are pushed to Docker Hub:

```text
Docker Hub
└── trushangdev
    ├── devops-poc-user-service
    ├── devops-poc-product-service
    └── devops-poc-order-service
```

The image naming convention is:

```text
trushangdev/devops-poc-<service>:<git-sha>
```

Example:

```text
trushangdev/devops-poc-user-service:9858175a...
```

### Core logic

The relationship is:

```text
Git Commit
    ↓
GitHub Actions
    ↓
Docker Build
    ↓
Docker Image
    ↓
Docker Hub
    ↓
GitOps values.yaml
    ↓
Argo CD
    ↓
Kubernetes Pod
```

For example:

```text
Commit: 9858175a...
        ↓
Image:
trushangdev/devops-poc-user-service:9858175a...
        ↓
GitOps:
image.tag = 9858175a...
        ↓
Kubernetes:
Pod runs that exact image
```

This gives us **end-to-end traceability**:

```text
Running Pod
    ↓
Docker Image
    ↓
Git SHA
    ↓
Application Source Code
```

### Important factor to remember

**Don't use `latest` for this GitOps deployment model.**

With:

```text
user-service:latest
```

the tag can point to different images over time.

With:

```text
user-service:9858175a...
```

the deployment points to a specific application version.

So:

> **Git SHA = immutable version reference**

Also remember the difference between the two Git repositories:

```text
devops-poc-app
    ↓
produces image

devops-poc-gitops
    ↓
declares which image Kubernetes should run
```

### Our M10 → M11 connection

M10 creates/publishes the image.

M11 gives that image a proper registry location and immutable version.

```text
M10
CI automation
     ↓
M11
Docker Hub + SHA image
     ↓
M9/M13
GitOps + Argo CD
     ↓
Kubernetes
```

**Mental shortcut:**

> **M10 = Build & automate.
> M11 = Store & version the artifact.**
