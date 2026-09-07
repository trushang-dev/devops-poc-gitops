# DevOps POC — Observability

This document describes the observability architecture implemented in the DevOps POC and explains how application metrics flow from the microservices into Prometheus and Grafana, with OpenTelemetry Collector included in the telemetry stack.

The observability stack is designed to answer three operational questions:

1. **Is the application running?**
2. **How is the application behaving?**
3. **Can we identify and investigate abnormal behavior?**

---

# 1. Observability Stack

The project uses:

| Component | Purpose |
|---|---|
| Prometheus | Metrics collection and time-series storage |
| Grafana | Metrics visualization and dashboards |
| OpenTelemetry Collector | Telemetry collection/processing component |
| Kubernetes | Runtime environment and workload metadata |
| Grafana Dashboard ConfigMaps | Declarative dashboard definitions |

The application consists of:

```text
user-service
product-service
order-service
````

These workloads run inside Kubernetes.

---

# 2. High-Level Architecture

```text
                    Kubernetes Cluster
                           |
          ┌────────────────┼────────────────┐
          |                |                |
          v                v                v
    user-service    product-service    order-service
          |                |                |
          └────────────────┼────────────────┘
                           |
                           v
                     Application
                       Metrics
                           |
             ┌─────────────┴─────────────┐
             |                           |
             v                           v
       Prometheus              OpenTelemetry Collector
             |                           |
             |                           |
             └─────────────┬─────────────┘
                           |
                           v
                       Grafana
                           |
                           v
                      Dashboards
```

The exact telemetry path depends on the configured scrape and Collector pipelines.

The important operational distinction is that:

```text
Prometheus
    =
Metrics collection / storage

Grafana
    =
Visualization

OpenTelemetry Collector
    =
Telemetry collection / processing
```

---

# 3. Observability vs Health Checks

Kubernetes health checks and observability solve different problems.

## Kubernetes probes

### Readiness

Determines whether a Pod should receive application traffic.

```text
Readiness failure
       |
       v
Pod becomes NotReady
       |
       v
Removed from usable Service endpoints
```

### Liveness

Determines whether the container should be restarted.

```text
Liveness failure
       |
       v
Container restart
```

## Observability

Metrics and dashboards help answer:

```text
How many requests are arriving?
How many are failing?
How long do requests take?
How many Pods are running?
Are Pods restarting?
Is the application scaling?
```

Therefore:

```text
Probes
  =
Runtime health decisions

Metrics
  =
Operational visibility
```

---

# 4. Prometheus

Prometheus is the primary metrics system used by the monitoring stack.

It collects time-series metrics and makes them available for querying.

The project uses Prometheus to observe application and Kubernetes-related metrics.

Typical metrics can be used to understand:

* Request volume
* Application availability
* HTTP behavior
* Pod state
* Container resource usage
* Restart activity
* Kubernetes workload state
* HPA behavior

---

# 5. Prometheus Scraping

Application metrics are scraped via a `PodMonitor`, one per service — not a `ServiceMonitor`:

```text
Application
     |
     | metrics
     v
Pod (discovered directly by IP)
     |
     v
PodMonitor
     |
     v
Prometheus
     |
     v
Time-Series Data
```

This is a deliberate choice, not an arbitrary one: scraping the Service's ClusterIP instead of each Pod directly let `kube-proxy` load-balance every scrape across replicas, and since each replica keeps its own independent in-memory counter, the resulting series jumped between unrelated counter values instead of increasing monotonically (§16 covers this in detail). A `PodMonitor` scrapes every matching Pod as its own stable target, which avoids that entirely.

Check PodMonitors:

```bash
kubectl get podmonitors -n devops-poc
```

Inspect a PodMonitor:

```bash
kubectl describe podmonitor <name> -n devops-poc
```

Check application Pods (a PodMonitor targets Pods directly, not the Service):

```bash
kubectl get pods -n devops-poc -o wide
```

Check endpoints (still useful for confirming a Pod is Ready, even though PodMonitor doesn't scrape through the Service):

```bash
kubectl get endpoints -n devops-poc
```

Note: the OpenTelemetry Collector *does* have its own `ServiceMonitor` (for its own self-telemetry, not application metrics) — see §15.

---

# 6. Debugging the Metrics Pipeline

When application metrics are missing, debug from the source outward.

```text
Application
     |
     v
Metrics endpoint
     |
     v
Pod (discovered directly)
     |
     v
PodMonitor
     |
     v
Prometheus
     |
     v
Grafana
```

Do not start with Grafana if Prometheus is not receiving the metrics.

---

# 7. Application Metrics

The application exposes metrics that can be used to understand application behavior.

The dashboard currently provides an application-level view including:

* Service availability
* Application instances
* Request activity
* Service distribution
* Pod restart information

The exact metric names and queries should be taken from the current dashboard definitions rather than assumed from generic Prometheus examples.

---

# 8. Grafana

Grafana provides the visualization layer for the monitoring stack.

It queries Prometheus and displays the resulting metrics through dashboards.

Conceptually:

```text
Prometheus
     |
     | PromQL
     v
Grafana
     |
     v
Dashboard Panels
```

Grafana does not replace Prometheus.

It is primarily the visualization and dashboard layer.

---

# 9. Application Overview Dashboard

The project includes an Application Overview dashboard:

```text
kubernetes/monitoring/
└── grafana-dashboard-application-overview.json
```

The dashboard provides a high-level view of the three application services:

```text
user-service
product-service
order-service
```

The dashboard includes visibility into areas such as:

* Service availability
* Application instance count
* Request activity
* Service distribution
* Pod restart activity

The dashboard is intended as an operational overview rather than a complete application-performance monitoring solution.

---

# 10. Grafana Dashboard Provisioning

Dashboards are stored declaratively in the GitOps repository.

The repository contains dashboard JSON and Kubernetes ConfigMaps:

```text
kubernetes/monitoring/
├── grafana-dashboard-application-overview.json
├── grafana-dashboard-application-overview-configmap.yaml
├── grafana-dashboard-devops-poc.json
└── grafana-dashboard-configmap.yaml
```

The conceptual provisioning flow is:

```text
GitOps Repository
       |
       v
Kubernetes ConfigMap
       |
       | grafana_dashboard=1
       v
Grafana Dashboard Sidecar
       |
       v
Dashboard JSON
       |
       v
Grafana
       |
       v
Dashboard
```

The Grafana sidecar watches Kubernetes resources matching the configured dashboard label.

---

# 11. Dashboard ConfigMaps

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

The dashboard label is important because it identifies ConfigMaps that should be processed by the Grafana dashboard sidecar.

---

# 12. Grafana Sidecar

The monitoring stack uses a Grafana dashboard sidecar.

Its role is to watch Kubernetes resources containing dashboard definitions and make those dashboards available to Grafana.

Conceptually:

```text
Kubernetes API
      |
      | WATCH
      v
Grafana Sidecar
      |
      v
Dashboard JSON
      |
      v
Grafana
```

This allows dashboards to be managed as configuration rather than manually created only through the Grafana UI.

---

# 13. GitOps and Dashboards

Dashboard definitions are stored in Git.

This means dashboard configuration can be reviewed and versioned alongside the rest of the infrastructure.

The conceptual flow is:

```text
Developer
    |
    v
GitOps Repository
    |
    v
Kubernetes ConfigMap
    |
    v
Grafana Sidecar
    |
    v
Grafana Dashboard
```

This provides version control for dashboard definitions.

### Important

Dashboard provisioning and application deployment are related but are not the same operation.

A change to an application image does not inherently require a dashboard change.

Likewise, changing a dashboard definition does not require rebuilding the application container.

---

# 14. Argo CD and Observability

Argo CD manages GitOps reconciliation for resources that belong to its configured Application.

The observability stack may also contain resources installed or managed through Helm.

For example, the monitoring stack uses:

```text
kube-prometheus-stack
```

and the project contains monitoring Helm configuration under:

```text
helm/monitoring/
```

Do not assume every monitoring resource is directly managed by the application Argo CD Application.

Ownership should be verified from the live Kubernetes metadata.

Useful command:

```bash
kubectl get configmaps -n devops-monitoring \
  -l grafana_dashboard=1 \
  -o custom-columns='NAME:.metadata.name,MANAGED-BY:.metadata.labels.app\.kubernetes\.io/managed-by,ARGOCD:.metadata.labels.argocd\.argoproj\.io/instance'
```

This helps determine whether a resource is:

* Helm-managed
* Argo CD-managed
* managed through another mechanism

---

# 15. OpenTelemetry Collector

OpenTelemetry provides a vendor-neutral framework for collecting and processing telemetry.

The project includes OpenTelemetry Collector configuration:

```text
helm/monitoring/otel-collector-values.yaml
```

The Collector can be inspected with:

```bash
kubectl get pods -n devops-monitoring | grep otel
```

Inspect logs:

```bash
kubectl logs <otel-pod> -n devops-monitoring
```

Inspect monitoring configuration:

```bash
kubectl get configmaps -n devops-monitoring
```

---

# 16. Avoiding Duplicate Metrics

One of the lessons from this POC was that telemetry pipelines must be understood as complete data flows.

It is possible for the same metric to be collected through multiple paths.

For example:

```text
Application
    |
    +----------> Prometheus
    |
    +----------> OpenTelemetry Collector
```

If both pipelines collect and expose the same metric into the same monitoring system, duplicate series or unexpected values can occur.

When investigating duplicated metrics, identify:

```text
1. Metric source
2. Scrape configuration
3. PodMonitor (application metrics) / ServiceMonitor (Collector self-telemetry)
4. OTel Collector pipeline
5. Prometheus target
6. Grafana query
```

Do not assume the problem is a Grafana visualization issue.

---

# 17. Useful Kubernetes Commands

## Check monitoring Pods

```bash
kubectl get pods -n devops-monitoring
```

## Check monitoring Services

```bash
kubectl get svc -n devops-monitoring
```

## Check monitoring resources

```bash
kubectl get all -n devops-monitoring
```

## Check PodMonitors (application metrics)

```bash
kubectl get podmonitors -n devops-poc
```

## Check ServiceMonitors (e.g. the OTel Collector's own self-telemetry)

```bash
kubectl get servicemonitors -A
```

## Check dashboard ConfigMaps

```bash
kubectl get configmaps -n devops-monitoring \
  -l grafana_dashboard=1
```

## Check recent monitoring events

```bash
kubectl get events -n devops-monitoring \
  --sort-by=.lastTimestamp
```

---

# 18. Grafana Troubleshooting

If Grafana is unavailable:

```bash
kubectl get pods -n devops-monitoring | grep grafana
```

Inspect the Pod:

```bash
kubectl describe pod <grafana-pod> -n devops-monitoring
```

Check logs:

```bash
kubectl logs <grafana-pod> -n devops-monitoring
```

If the Pod has restarted:

```bash
kubectl logs <grafana-pod> \
  -n devops-monitoring \
  --previous
```

Check dashboard ConfigMaps:

```bash
kubectl get configmaps -n devops-monitoring \
  -l grafana_dashboard=1
```

For dashboard-specific problems, see:

```text
TROUBLESHOOTING.md
```

---

# 19. Prometheus Troubleshooting

Check Prometheus:

```bash
kubectl get pods -n devops-monitoring | grep prometheus
```

Check Prometheus logs:

```bash
kubectl logs <prometheus-pod> -n devops-monitoring
```

Check PodMonitors (application metrics use these, not ServiceMonitors):

```bash
kubectl get podmonitors -n devops-poc
```

Check application endpoints:

```bash
kubectl get endpoints -n devops-poc
```

If the application has no healthy endpoints, Prometheus may also be unable to collect metrics through that Service.

---

# 20. Observability During Deployment

During a normal application deployment:

```text
New Image
    |
    v
Deployment
    |
    v
RollingUpdate
    |
    v
New Pods
    |
    v
Readiness
    |
    v
Service
    |
    v
Metrics
    |
    v
Prometheus
    |
    v
Grafana
```

The monitoring system should therefore provide visibility into the effect of a deployment.

Useful things to observe include:

* Pod availability
* Restart count
* Request volume
* Error behavior
* Resource consumption
* Replica count
* HPA activity

---

# 21. Observability During Failure Testing

The project includes failure testing such as:

* Pod deletion
* Readiness failure
* Liveness failure
* Deployment rollout
* HPA behavior

Observability helps show the difference between:

```text
Expected recovery
```

and:

```text
Unexpected failure
```

For example:

```text
Pod deleted
    ↓
ReplicaSet creates replacement
    ↓
New Pod starts
    ↓
Readiness succeeds
    ↓
Service endpoint restored
```

Metrics and Kubernetes events can be used together to verify this recovery.

---

# 22. Operational Dashboard Improvements

The current Application Overview dashboard provides a useful high-level view.

For a more production-oriented dashboard, additional panels could include:

### Request Rate

```text
Requests / second
```

### Error Rate

```text
5xx requests / total requests
```

### Latency

```text
p50
p95
p99
```

### Resource Usage

```text
CPU
Memory
```

### HPA

```text
Desired replicas
Current replicas
CPU utilization
```

### Pod Restarts

Prefer observing the **rate/change in restarts** rather than only the cumulative restart count.

These are improvement opportunities, not claims that all of these panels are currently implemented.

---

# 23. Observability Ownership

Different components are responsible for different functions.

| Component               | Responsibility                         |
| ----------------------- | -------------------------------------- |
| Kubernetes              | Workload state and recovery            |
| Deployment              | Desired application replicas / rollout |
| HPA                     | Runtime replica scaling                |
| Prometheus              | Metrics collection and storage         |
| OpenTelemetry Collector | Telemetry collection/processing        |
| Grafana                 | Visualization                          |
| Grafana Sidecar         | Dashboard provisioning                 |
| Argo CD                 | GitOps reconciliation                  |

This distinction is important during troubleshooting.

For example:

```text
Pod deleted
    →
Kubernetes handles recovery
```

while:

```text
Git configuration changed
    →
Argo CD handles reconciliation
```

and:

```text
Metric visualization
    →
Grafana queries Prometheus
```

---

# 24. Observability Verification Checklist

After deploying the monitoring stack, verify:

```text
[ ] Prometheus Pod is healthy
[ ] Grafana Pod is healthy
[ ] OpenTelemetry Collector is healthy
[ ] Application Pods are Ready
[ ] Application Services have endpoints
[ ] PodMonitors exist as expected (one per service)
[ ] Prometheus receives expected metrics
[ ] Grafana can query Prometheus
[ ] Dashboard ConfigMaps exist
[ ] Grafana dashboards are visible
[ ] Dashboard data is updating
```

Useful commands:

```bash
kubectl get pods -n devops-monitoring

kubectl get svc -n devops-monitoring

kubectl get podmonitors -n devops-poc

kubectl get configmaps -n devops-monitoring \
  -l grafana_dashboard=1

kubectl get endpoints -n devops-poc
```

---

# 25. Known Limitations

This observability implementation is a learning POC rather than a production monitoring platform.

Known limitations include:

* Local Minikube environment
* Limited application-level alerting
* Dashboard coverage is intentionally focused
* No production-grade notification/incident-management integration
* No long-term remote metrics storage
* No multi-cluster observability
* No production HA monitoring architecture
* OpenTelemetry pipeline complexity requires careful validation to avoid duplicate collection
* metrics-server is a separate Kubernetes dependency and is not equivalent to Prometheus

The project intentionally documents these limitations rather than presenting the local monitoring stack as production-ready.

---

# 26. Key Observability Lessons

### 1. Metrics are only useful if the collection path is understood

```text
Source
  ↓
Collection
  ↓
Storage
  ↓
Query
  ↓
Visualization
```

### 2. Grafana is not the metrics source

Grafana visualizes data obtained from a datasource such as Prometheus.

### 3. Kubernetes probes are not monitoring

Readiness and liveness determine runtime behavior.

Metrics provide operational visibility.

### 4. Controllers have different responsibilities

Kubernetes controllers handle workload behavior.

Argo CD handles Git-to-cluster reconciliation.

### 5. Declarative dashboards are easier to reproduce

Dashboard JSON stored in Git can be reviewed, versioned, and recreated.

### 6. Duplicate telemetry usually means overlapping collection paths

When a metric appears twice, inspect the entire telemetry pipeline rather than only the dashboard query.

### 7. Observability should help answer operational questions

A dashboard should not exist simply because metrics are available.

It should help answer:

```text
Is the service healthy?

Is traffic increasing?

Are errors increasing?

Are requests becoming slower?

Are Pods restarting?

Is HPA scaling?

Did the latest deployment affect application behavior?
```

---

# 27. Related Documentation

| Document                                                    | Purpose                                 |
| ----------------------------------------------------------- | --------------------------------------- |
| `README.md`                                                 | Project overview                        |
| `ARCHITECTURE.md`                                           | System architecture                     |
| `PROJECT_JOURNEY.md`                                        | Project evolution                       |
| `RUNBOOK.md`                                                | Operational procedures                  |
| `TROUBLESHOOTING.md`                                        | Failure investigation                   |
| `SECURITY.md`                                               | Security practices                      |
| `M12_Observability: Prometheus, Grafana & OpenTelemetry.md` | Historical observability implementation |
| `M13_Argo CD & GitOps Reconciliation.md`                    | GitOps reconciliation                   |
| `PROJECT_JOURNEY.md` (Phase 8)                              | Failure testing                         |
| `PROJECT_JOURNEY.md` (Phase 9)                              | End-to-end DevSecOps validation         |

This document is deliberately different in purpose from the M12 milestone doc: `M12_Observability...` answers "what did we build and what problems did we encounter?"; this document answers "how does observability work now, and how do I operate/debug it?"

Note also that not every Grafana dashboard ConfigMap is managed by the `devops-poc` Argo CD Application — some are Helm-managed independently of it. Verify ownership from live Kubernetes metadata (§14) rather than assuming Argo CD manages everything in `devops-monitoring`.
