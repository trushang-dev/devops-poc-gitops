# M12 — Observability: Prometheus, Grafana & OpenTelemetry

### What is it used for

* Monitor the **health and performance** of the Kubernetes application.
* Collect metrics from Kubernetes and application workloads.
* Visualize metrics through Grafana dashboards.
* Provide visibility into CPU, memory, Pod health, request/application metrics, and infrastructure state.

### Key points to review / make strong

* Metrics vs Logs vs Traces
* Prometheus
* Grafana
* OpenTelemetry (OTel)
* OTel Collector
* Prometheus scraping
* `ServiceMonitor` vs `PodMonitor`
* Prometheus targets
* PromQL basics
* Grafana datasource
* Kubernetes metrics
* `kube-state-metrics`
* Node Exporter
* Alerting basics
* Why observability is different from simply checking `kubectl get pods`

### How it is implemented in our POC

Our monitoring stack runs in:

```text
devops-monitoring
```

The main components are:

```text
devops-monitoring
│
├── Prometheus
├── Grafana
├── OpenTelemetry Collector
├── kube-state-metrics
├── Node Exporter
└── Prometheus Operator
```

Conceptually:

```text
Kubernetes / Application
          │
          │ metrics
          ▼
      Prometheus
          │
          │ PromQL
          ▼
       Grafana
          │
          ▼
      Dashboard
```

### Core logic — Prometheus

Prometheus primarily works using a **pull/scrape model**.

```text
Application / Exporter
        ↑
        │ scrape metrics
        │
    Prometheus
        │
        ▼
   Time-series DB
```

For our application Pods, the monitoring configuration allows Prometheus to discover and scrape the relevant metrics.

We verified that the application Pods are being scraped:

```text
up{job=~"devops-poc.*"} == 1
```

So:

```text
Pod
 ↓
Metrics endpoint
 ↓
Prometheus
 ↓
Stored time-series
```

### Core logic — Grafana

Grafana doesn't normally collect the metrics itself.

Instead:

```text
Prometheus
    ↓
Datasource
    ↓
Grafana
    ↓
PromQL queries
    ↓
Panels
    ↓
Dashboard
```

Our POC has a **DevOps POC Overview** dashboard with multiple monitoring panels.

So remember:

> **Prometheus collects/stores metrics; Grafana visualizes them.**

### Core logic — OpenTelemetry

OpenTelemetry is our **observability instrumentation/collection layer**.

Conceptually:

```text
Application / telemetry source
          ↓
   OpenTelemetry
          ↓
   OTel Collector
          ↓
 Observability backend
```

The OTel Collector provides a central place to receive/process/export telemetry.

A very important distinction:

> **OpenTelemetry is not the same thing as Prometheus or Grafana.**

Think:

```text
OpenTelemetry
    = telemetry standard + collection ecosystem

Prometheus
    = metrics collection/storage/querying

Grafana
    = visualization
```

### Kubernetes monitoring components

#### kube-state-metrics

Provides metrics about Kubernetes object state.

For example:

```text
Deployment replicas
Pod status
HPA state
Node/Workload state
```

It answers questions like:

> "What does Kubernetes believe the state of this object is?"

#### Node Exporter

Provides host/node-level metrics such as:

```text
CPU
Memory
Disk
Network
```

So:

```text
Node Exporter
      ↓
Node infrastructure metrics
```

### Important factor to remember

Don't reduce observability to:

```text
CPU graph = monitoring
```

A useful mental model is:

```text
              OBSERVABILITY
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
     Metrics      Logs       Traces
       │
       ▼
   Prometheus
       │
       ▼
    Grafana
```

Our current POC is particularly focused on **metrics/monitoring**.

### Our POC mental model

```text
                    Kubernetes
                        │
        ┌───────────────┼────────────────┐
        │               │                │
   App Pods        Kubernetes         Nodes
        │             State              │
        │               │                │
        └───────┬───────┘                │
                ↓                        ↓
           Prometheus              Node Exporter
                │
                │
                ├──── kube-state-metrics
                │
                └──── Application metrics
                         │
                         ▼
                      Grafana
                         │
                         ▼
                    Dashboard
```

**Mental shortcut:**

> **Prometheus = collect/query metrics**
> **Grafana = visualize metrics**
> **OpenTelemetry = collect/process telemetry**
> **kube-state-metrics = Kubernetes object state**
> **Node Exporter = node metrics**
