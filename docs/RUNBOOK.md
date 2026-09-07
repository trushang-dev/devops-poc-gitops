# DevOps POC — Operations Runbook

This runbook provides the operational procedures for running, deploying, validating, monitoring, troubleshooting, and rolling back the DevOps POC.

The project uses:

- Kubernetes / Minikube
- Helm
- Argo CD
- GitHub Actions
- Docker Hub
- Prometheus
- Grafana
- OpenTelemetry Collector

---

## 1. Architecture at a Glance

```text
Developer
    |
    v
GitHub App Repository
    |
    | Pull Request / Release
    v
GitHub Actions
    |
    | Build + Test + Trivy Scan
    v
Docker Hub
    |
    | Image
    v
GitOps Repository
    |
    | Git change
    v
Argo CD
    |
    | Reconciliation
    v
Kubernetes / Minikube
    |
    +--> Deployments
    |       |
    |       +--> Pods
    |
    +--> Services
    |
    +--> Ingress
    |
    +--> HPA
    |
    +--> ConfigMaps / Secrets
    |
    +--> Prometheus
    |
    +--> Grafana
    |
    +--> OpenTelemetry Collector
````

---

# 2. Prerequisites

The local environment requires the following tools:

```bash
docker
kubectl
minikube
helm
git
```

Optional / operational tools:

```bash
curl
jq
```

Verify individual tools:

```bash
docker --version
kubectl version --client
minikube version
helm version
git --version
```

---

# 3. Start the Local Kubernetes Environment

Start Minikube using Docker:

```bash
minikube start --driver=docker
```

Verify the cluster:

```bash
minikube status
```

Expected:

```text
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

Verify the Kubernetes nodes:

```bash
kubectl get nodes
```

---

# 4. Check Cluster Health

Check all namespaces:

```bash
kubectl get namespaces
```

Check application resources:

```bash
kubectl get all -n devops-poc
```

Check monitoring resources:

```bash
kubectl get all -n devops-monitoring
```

Check Argo CD:

```bash
kubectl get pods -n argocd
```

A healthy environment should have the major application and platform components in `Running` or otherwise expected healthy states.

---

# 5. Application Deployment

The application is managed through the GitOps repository.

The primary deployment configuration is located under:

```text
helm/microservices/
```

The Helm chart manages the three application services:

```text
user-service
product-service
order-service
```

Do not manually modify running Kubernetes resources as the normal deployment mechanism.

The intended deployment flow is:

```text
Application Code
      |
      v
GitHub Actions
      |
      v
Container Image
      |
      v
GitOps Repository
      |
      v
Argo CD
      |
      v
Kubernetes
```

---

# 6. Verify Application Deployments

Check deployments:

```bash
kubectl get deployments -n devops-poc
```

Check ReplicaSets:

```bash
kubectl get replicasets -n devops-poc
```

Check Pods:

```bash
kubectl get pods -n devops-poc -o wide
```

Check Services:

```bash
kubectl get services -n devops-poc
```

Check Ingress:

```bash
kubectl get ingress -n devops-poc
```

---

# 7. Verify Pod Health

Inspect Pod status:

```bash
kubectl get pods -n devops-poc
```

For a specific Pod:

```bash
kubectl describe pod <pod-name> -n devops-poc
```

View logs:

```bash
kubectl logs <pod-name> -n devops-poc
```

Follow logs:

```bash
kubectl logs -f <pod-name> -n devops-poc
```

View logs for a Deployment:

```bash
kubectl logs deployment/user-service -n devops-poc
```

---

# 8. Verify Services

List application Services:

```bash
kubectl get svc -n devops-poc
```

Inspect a Service:

```bash
kubectl describe svc user-service -n devops-poc
```

Check Service endpoints:

```bash
kubectl get endpoints -n devops-poc
```

A Service should have healthy backend endpoints when the corresponding Pods are Ready.

---

# 9. Verify Ingress

Check the configured Ingress:

```bash
kubectl get ingress -n devops-poc
```

Inspect it:

```bash
kubectl describe ingress <ingress-name> -n devops-poc
```

If Minikube ingress is required:

```bash
minikube addons enable ingress
```

Check the ingress controller:

```bash
kubectl get pods -n ingress-nginx
```

---

# 10. Verify Application Endpoints

All three services expose a `/health` endpoint. `user-service` additionally exposes `/version` (the other two do not have an equivalent endpoint).

Example:

```bash
kubectl exec deployment/user-service -n devops-poc -- \
  wget -qO- http://localhost:3001/health
```

Version (`user-service` only):

```bash
kubectl exec deployment/user-service -n devops-poc -- \
  wget -qO- http://localhost:3001/version
```

Repeat the `/health` check for `product-service` (port 3002) and `order-service` (port 3003).

This provides a direct verification path that does not depend on Ingress routing.

---

# 11. Check Argo CD

Check Argo CD Pods:

```bash
kubectl get pods -n argocd
```

Check the Application:

```bash
kubectl get application devops-poc -n argocd
```

Get a concise status:

```bash
kubectl get application devops-poc -n argocd \
  -o jsonpath='{.status.sync.status}{" "}{.status.health.status}{"\n"}'
```

Expected healthy state:

```text
Synced Healthy
```

---

# 12. Inspect Argo CD Application Resources

List resources managed by the Application:

```bash
kubectl get application devops-poc -n argocd \
  -o jsonpath='{range .status.resources[*]}{.kind}{"\t"}{.namespace}{"\t"}{.name}{"\t"}{.status}{"\n"}{end}'
```

This is useful when determining whether a Kubernetes resource is actually managed by Argo CD.

---

# 13. GitOps Deployment

The normal deployment process is:

```text
1. Change application code
        |
2. Create Pull Request
        |
3. CI validates the change
        |
4. Release branch is created
        |
5. Images are built
        |
6. Trivy scans images
        |
7. Images are pushed to Docker Hub
        |
8. GitOps repository image tag is updated
        |
9. Argo CD detects Git change
        |
10. Argo CD synchronizes
        |
11. Kubernetes performs rollout
        |
12. New Pods become Ready
```

The GitOps repository is the desired-state source.

---

# 14. Verify a GitOps Release

After a release, check:

```bash
git log --oneline -5
```

in the GitOps repository.

Then check Argo:

```bash
kubectl get application devops-poc -n argocd
```

Check deployments:

```bash
kubectl get deployments -n devops-poc
```

Check Pods:

```bash
kubectl get pods -n devops-poc
```

Verify the deployed application version:

```bash
kubectl exec deployment/user-service -n devops-poc -- \
  wget -qO- http://localhost:3001/version
```

Repeat for the other services when required.

---

# 15. Monitor a Rolling Deployment

Watch the deployment:

```bash
kubectl rollout status deployment/user-service -n devops-poc
```

Watch Pods:

```bash
kubectl get pods -n devops-poc -w
```

Inspect rollout history:

```bash
kubectl rollout history deployment/user-service -n devops-poc
```

Inspect the current Deployment:

```bash
kubectl describe deployment user-service -n devops-poc
```

---

# 16. Kubernetes-Native Rollback

Kubernetes maintains Deployment rollout history.

View revisions:

```bash
kubectl rollout history deployment/user-service -n devops-poc
```

Rollback:

```bash
kubectl rollout undo deployment/user-service -n devops-poc
```

Monitor the rollback:

```bash
kubectl rollout status deployment/user-service -n devops-poc
```

Verify:

```bash
kubectl get pods -n devops-poc
```

Verify the deployed image:

```bash
kubectl get deployment user-service -n devops-poc \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### Important

Kubernetes-native rollback changes the live Kubernetes Deployment.

It does **not** change the GitOps repository.

Therefore, this should be treated as an operational rollback mechanism rather than the preferred long-term GitOps correction.

---

# 17. GitOps Rollback

The preferred GitOps rollback mechanism is to revert the Git change.

Example:

```bash
git log --oneline
```

Identify the commit that introduced the unwanted deployment state.

Revert it:

```bash
git revert <commit>
```

Push the change:

```bash
git push origin main
```

Argo CD detects the Git change and reconciles Kubernetes back to the desired state.

Verify:

```bash
kubectl get application devops-poc -n argocd
```

Expected:

```text
Synced Healthy
```

Then verify the Deployment:

```bash
kubectl get deployment user-service -n devops-poc \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

---

# 18. Test GitOps Drift Detection

To intentionally create drift:

```bash
kubectl set env deployment/user-service \
  DEPLOY_NOTE=manual-drift-test \
  -n devops-poc
```

Inspect the live value:

```bash
kubectl get deployment user-service -n devops-poc \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="DEPLOY_NOTE")].value}{"\n"}'
```

Argo CD should detect that the live state differs from Git.

With automated self-healing enabled, Argo CD reconciles the resource back to the Git-defined state.

Verify:

```bash
kubectl get deployment user-service -n devops-poc \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="DEPLOY_NOTE")].value}{"\n"}'
```

---

# 19. Failure Testing

## Pod Failure

Delete a Pod:

```bash
kubectl delete pod <pod-name> -n devops-poc
```

Watch recovery:

```bash
kubectl get pods -n devops-poc -w
```

The ReplicaSet/Deployment controller should recreate the Pod.

This recovery is performed by Kubernetes, not Argo CD.

---

## Readiness Failure

When a Pod fails its readiness probe:

```text
Pod
 ↓
Readiness = false
 ↓
Removed from Service endpoints
 ↓
Traffic stops
```

Inspect:

```bash
kubectl describe pod <pod-name> -n devops-poc
```

Check endpoints:

```bash
kubectl get endpoints -n devops-poc
```

---

## Liveness Failure

When a Pod fails its liveness probe:

```text
Liveness failure
       ↓
Container restart
       ↓
Application starts again
       ↓
Readiness check
       ↓
Traffic restored
```

Inspect restart counts:

```bash
kubectl get pods -n devops-poc
```

Detailed information:

```bash
kubectl describe pod <pod-name> -n devops-poc
```

---

# 20. Horizontal Pod Autoscaling

Check HPA:

```bash
kubectl get hpa -n devops-poc
```

Detailed information:

```bash
kubectl describe hpa <hpa-name> -n devops-poc
```

Watch replica changes:

```bash
kubectl get pods -n devops-poc -w
```

The HPA controls the desired replica count based on configured metrics.

Do not manually change the Deployment replica count when HPA owns scaling.

---

# 21. Prometheus

Check monitoring namespace:

```bash
kubectl get pods -n devops-monitoring
```

Check Prometheus:

```bash
kubectl get pods -n devops-monitoring | grep prometheus
```

Application metrics are scraped via a per-service `PodMonitor` (not a `ServiceMonitor` — scraping through the Service was tried first and produced corrupted counters, since `kube-proxy` load-balances each scrape across replicas; see `docs/OBSERVABILITY.md`):

```bash
kubectl get podmonitors -n devops-poc
```

Inspect a PodMonitor:

```bash
kubectl describe podmonitor <name> -n devops-poc
```

Prometheus should discover application metrics according to the configured PodMonitor and scrape configuration.

---

# 22. Grafana

Check Grafana:

```bash
kubectl get pods -n devops-monitoring | grep grafana
```

Check Grafana Service:

```bash
kubectl get svc -n devops-monitoring | grep grafana
```

Application dashboards are provisioned using Kubernetes ConfigMaps and the Grafana dashboard sidecar.

Check dashboard ConfigMaps:

```bash
kubectl get configmaps -n devops-monitoring \
  -l grafana_dashboard=1
```

Inspect a dashboard ConfigMap:

```bash
kubectl describe configmap <dashboard-configmap> \
  -n devops-monitoring
```

---

# 23. OpenTelemetry Collector

Check Collector Pods:

```bash
kubectl get pods -n devops-monitoring | grep otel
```

Check Collector logs:

```bash
kubectl logs <otel-pod> -n devops-monitoring
```

If telemetry appears duplicated or missing, inspect the Collector configuration and Prometheus scrape configuration.

---

# 24. Check Events

Kubernetes events are often the fastest way to understand an operational problem.

```bash
kubectl get events -n devops-poc --sort-by=.lastTimestamp
```

For monitoring:

```bash
kubectl get events -n devops-monitoring --sort-by=.lastTimestamp
```

For Argo CD:

```bash
kubectl get events -n argocd --sort-by=.lastTimestamp
```

---

# 25. Common Troubleshooting Sequence

When something is broken, follow this order:

```text
1. Is Kubernetes running?
        ↓
2. Are Pods running?
        ↓
3. Are Pods Ready?
        ↓
4. Are containers restarting?
        ↓
5. Are Services configured?
        ↓
6. Are Service endpoints populated?
        ↓
7. Is Ingress routing correctly?
        ↓
8. Is Argo CD Synced?
        ↓
9. Is the application healthy?
        ↓
10. Are Prometheus/Grafana healthy?
```

Useful commands:

```bash
minikube status

kubectl get pods -n devops-poc

kubectl get svc -n devops-poc

kubectl get endpoints -n devops-poc

kubectl get ingress -n devops-poc

kubectl get application devops-poc -n argocd

kubectl get events -n devops-poc --sort-by=.lastTimestamp
```

For detailed troubleshooting, see:

```text
docs/TROUBLESHOOTING.md
```

---

# 26. Check Disk Usage

This POC runs locally using Docker and Minikube, so disk usage can become an operational constraint.

Check filesystem usage:

```bash
df -h
```

Check Docker disk usage:

```bash
docker system df
```

Check Minikube status:

```bash
minikube status
```

### Important

Do not immediately run:

```bash
docker system prune -a
```

The project intentionally uses multiple image versions for deployment and rollback testing.

Inspect Docker usage first and remove only resources that are known to be unnecessary.

---

# 27. Stop the Environment

To stop Minikube while preserving the cluster:

```bash
minikube stop
```

Check:

```bash
minikube status
```

---

# 28. Start the Existing Environment Again

```bash
minikube start --driver=docker
```

Verify:

```bash
minikube status
kubectl get nodes
kubectl get pods -A
```

---

# 29. Delete the Local Cluster

Only use this when the local cluster can be recreated.

```bash
minikube delete
```

This removes the Minikube cluster and its local Kubernetes state.

After deletion, the environment must be bootstrapped again.

---

# 30. Operational Golden Path

For normal development and demonstration, use:

```text
┌───────────────────────┐
│ Change Application    │
│ Code                  │
└───────────┬───────────┘
            ↓
┌───────────────────────┐
│ Pull Request          │
└───────────┬───────────┘
            ↓
┌───────────────────────┐
│ GitHub Actions        │
│ Test + Scan + Build   │
└───────────┬───────────┘
            ↓
┌───────────────────────┐
│ Docker Hub            │
└───────────┬───────────┘
            ↓
┌───────────────────────┐
│ GitOps Repository     │
│ Image Tag Updated     │
└───────────┬───────────┘
            ↓
┌───────────────────────┐
│ Argo CD               │
│ Detect + Reconcile    │
└───────────┬───────────┘
            ↓
┌───────────────────────┐
│ Kubernetes            │
│ RollingUpdate         │
└───────────┬───────────┘
            ↓
┌───────────────────────┐
│ Prometheus + Grafana  │
│ Observe                │
└───────────────────────┘
```

---

# 31. Rollback Decision Guide

| Situation                  | Preferred Action                    |
| -------------------------- | ----------------------------------- |
| Bad Pod                    | Kubernetes recreates Pod            |
| Failed readiness           | Kubernetes removes Pod from traffic |
| Failed liveness            | Kubernetes restarts container       |
| Deployment rollout problem | `kubectl rollout undo`              |
| Bad GitOps deployment      | `git revert`                        |
| Configuration drift        | Argo CD self-heal                   |
| HPA scaling issue          | Inspect HPA + metrics               |
| Application metrics issue  | Inspect PodMonitor / Prometheus     |
| Dashboard issue            | Inspect Grafana ConfigMap / sidecar |
| Local environment issue    | Inspect Minikube / Docker           |

---

# 32. Operational Principles

This POC follows several important operational principles:

### Git is the desired state

The GitOps repository defines what Kubernetes should look like.

### Argo CD reconciles desired state

Argo CD continuously compares Git with the Kubernetes cluster and reconciles differences.

### Kubernetes owns runtime recovery

Pod recreation, container restart, rolling updates, and HPA scaling are Kubernetes responsibilities.

### CI and CD are separated

GitHub Actions handles:

```text
Build
Test
Security Scan
Image Publish
GitOps Update
```

Argo CD handles:

```text
Git → Kubernetes reconciliation
```

### Observability is separate from deployment

Prometheus, Grafana, and OpenTelemetry provide visibility into the running system. They do not replace Kubernetes or Argo CD.

---

# 33. Related Documentation

| Document                                      | Purpose                          |
| --------------------------------------------- | -------------------------------- |
| `README.md`                                   | Project overview                 |
| `ARCHITECTURE.md`                             | System architecture              |
| `PROJECT_JOURNEY.md`                          | Project evolution                |
| `docs/OBSERVABILITY.md`                       | Monitoring and observability     |
| `docs/TROUBLESHOOTING.md`                     | Problem investigation            |
| `docs/SECURITY.md`                            | Security practices               |
| `docs/M13_Argo CD & GitOps Reconciliation.md` | Argo CD implementation           |
| `PROJECT_JOURNEY.md` (Phase 7)                | Deployment and rollback evidence |
| `PROJECT_JOURNEY.md` (Phase 8)                | Failure testing evidence         |
| `PROJECT_JOURNEY.md` (Phase 9)                | End-to-end DevSecOps evidence    |

This runbook deliberately avoids exact URLs, passwords, Docker Hub credentials, or machine-specific paths — commands are written to stay reproducible without exposing environment-specific secrets.
