# M8 — Helm Packaging

### What is it used for

* Package Kubernetes manifests into a **reusable Helm chart**.
* Avoid maintaining separate YAML files for every environment/service variation.
* Manage Kubernetes configuration through **templates + values**.

### Key points to review / make strong

* Helm Chart
* `Chart.yaml`
* `values.yaml`
* Templates
* Helm release
* Helm rendering
* `helm install`
* `helm upgrade`
* `helm rollback`
* Values overriding templates
* Templating with `{{ ... }}`
* Why Helm is useful for Kubernetes applications

### How it is implemented in our POC

Our three microservices are packaged together:

```text
devops-poc-gitops
└── helm/
    └── microservices/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            ├── hpa.yaml
            ├── configmap.yaml
            ├── secret.yaml
            └── ...
```

Instead of hardcoding values directly into Kubernetes YAML, the chart uses:

```text
values.yaml
      ↓
Helm templates
      ↓
Rendered Kubernetes YAML
      ↓
Kubernetes
```

### Core logic

Think of Helm as a **Kubernetes YAML template engine + package/release manager**.

For example, instead of writing:

```yaml
replicas: 2
```

directly into every Deployment, the template can use:

```yaml
replicas: {{ .Values.replicaCount }}
```

and `values.yaml` contains the actual value.

So:

```text
values.yaml
    ↓
Helm template processing
    ↓
helm template / helm install / helm upgrade
    ↓
Kubernetes manifests
```

### Our image configuration

This became particularly important for our GitOps flow.

Our `values.yaml` contains:

```yaml
image:
  repository: trushangdev
  tag: "<git-sha>"
```

The Deployment template constructs the final image:

```text
trushangdev/
devops-poc-user-service:
<git-sha>
```

Similarly:

```text
trushangdev/devops-poc-product-service:<sha>
trushangdev/devops-poc-order-service:<sha>
```

This allows CI to update the **image tag in Git**, while Helm remains responsible for turning that desired configuration into Kubernetes resources.

### Important factor to remember

Don't think:

> Helm = Kubernetes.

Think:

```text
Kubernetes
    ↓
Runs/manages resources

Helm
    ↓
Packages + templates those resources
```

And in **our POC**, there is one more important layer:

```text
GitOps Repository
       ↓
Helm Chart
       ↓
Argo CD
       ↓
Kubernetes
```

Also remember an important architectural change after M13:

> **Argo CD now owns the Helm deployment/reconciliation.**

So we should **not manually run `helm upgrade` against the application as the normal deployment mechanism**. The desired Helm configuration lives in Git, and Argo CD reconciles it into Kubernetes.
