# M14 — Deployment Validation & Operational Verification

### What is it used for

* Verify that the application is **actually running correctly after deployment**.
* Confirm Kubernetes resources, application health, networking, autoscaling, and observability are working together.
* Catch deployment problems that a successful CI/CD pipeline alone cannot detect.

### Key points to review / make strong

* Deployment rollout status
* Pod readiness
* Service availability
* Ingress routing
* Health endpoints
* HPA status
* Application logs
* Prometheus metrics
* Grafana dashboards
* Argo CD `Synced` / `Healthy`
* Image verification
* Smoke testing
* Difference between **deployment success** and **application success**

### How it is implemented in our POC

After Argo CD deploys the application, we validate the complete stack:

```text
Git
 ↓
Argo CD
 ↓
Kubernetes Deployment
 ↓
Pods
 ↓
Service
 ↓
Ingress
 ↓
API
```

And separately:

```text
Pods
 ↓
Metrics
 ↓
Prometheus
 ↓
Grafana
```

### Core logic

A deployment isn't considered successful simply because:

```bash
kubectl apply
```

or Argo CD says:

```text
Synced
```

We verify multiple layers.

#### 1. Kubernetes

```bash
kubectl get deployments -n devops-poc
kubectl get pods -n devops-poc
```

Expected:

```text
READY = desired replicas
STATUS = Running
RESTARTS = 0
```

#### 2. Correct image

```bash
kubectl get pods -n devops-poc \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[*].image'
```

This confirms the Pods are actually running the image version declared by GitOps.

#### 3. Application health

Our POC exposes health endpoints through Ingress:

```text
/api/users/health
/api/products/health
/api/orders/health
```

Expected:

```text
HTTP 200
```

#### 4. HPA

Check:

```bash
kubectl get hpa -n devops-poc
```

This verifies that autoscaling is configured and the metrics pipeline is working.

#### 5. Argo CD

Check:

```text
Sync = Synced
Health = Healthy
```

This verifies:

```text
Git desired state
       =
Kubernetes state
```

#### 6. Observability

Prometheus should be able to scrape the application:

```text
up = 1
```

Grafana then provides the visual view of the collected metrics.

### Important factor to remember

**CI/CD success ≠ application success.**

Think in layers:

```text
Layer 1 → CI
          Did code build/test/scan?

Layer 2 → Image
          Was the correct image published?

Layer 3 → GitOps
          Does Git contain the correct desired version?

Layer 4 → Argo CD
          Did reconciliation happen?

Layer 5 → Kubernetes
          Are Pods healthy?

Layer 6 → Application
          Does the API actually respond?

Layer 7 → Observability
          Can we see/measure the running system?
```

### Mental shortcut

> **Build → Deploy → Verify → Observe**

That is the operational mindset you want to retain from M14.
