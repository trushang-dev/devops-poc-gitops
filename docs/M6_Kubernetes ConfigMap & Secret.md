# M6 — Kubernetes ConfigMap & Secret

### What is it used for

* Keep **configuration separate from application/container images**.
* Provide environment-specific configuration to Pods.
* Store sensitive values separately from normal configuration.

### Key points to review / make strong

* ConfigMap vs Secret
* Environment variables
* Mounting configuration as files
* Why configuration should not be hardcoded in Docker images
* Why secrets should not be committed as plaintext to Git
* Namespace scope
* Updating configuration and Pod behavior

### How it is implemented in our POC

For each microservice, we have configuration resources:

```text
devops-poc namespace
│
├── user-service
│   ├── ConfigMap
│   └── Secret
│
├── product-service
│   ├── ConfigMap
│   └── Secret
│
└── order-service
    ├── ConfigMap
    └── Secret
```

The configuration is consumed by the corresponding Deployment/Pods.

Conceptually:

```text
ConfigMap ───────┐
                 ├──→ Pod → Container → Application
Secret ──────────┘
```

### Core logic

The important separation is:

```text
Docker Image
    ↓
Application code + runtime
```

while:

```text
ConfigMap
    ↓
Normal configuration

Secret
    ↓
Sensitive configuration
```

This means the same Docker image can be used in different environments:

```text
             Same Image
                 │
        ┌────────┼────────┐
        ↓        ↓        ↓
       DEV      UAT      PROD
        │        │        │
    different configuration
```

### ConfigMap

Use it for **non-sensitive configuration**, such as:

```text
Environment
API configuration
Feature flags
Service configuration
Database host/name
```

### Secret

Use it for **sensitive values**, such as:

```text
Database password
API credentials
Tokens
Keys
```

One important nuance:

> Kubernetes `Secret` is **not automatically equivalent to strong encryption**. Its values are commonly base64-encoded in manifests; real production security also depends on how secrets are stored, RBAC, encryption at rest, external secret managers, etc.

### Important factor to remember

Think:

```text
Image
  = Application

ConfigMap
  = Configuration

Secret
  = Sensitive configuration
```

And don't confuse **Secret with Docker image secrets** or **GitHub Actions secrets**. They solve different problems at different stages:

```text
GitHub Actions Secret
        ↓
CI credentials

Kubernetes Secret
        ↓
Runtime application credentials
```

For our POC, M6 establishes the basic **configuration/secrets separation** needed before moving toward autoscaling, packaging, and GitOps.
