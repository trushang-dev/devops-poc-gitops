# M7 — Kubernetes HPA (Horizontal Pod Autoscaler)

### What is it used for

* Automatically increase or decrease the number of Pods based on **resource utilization**.
* Handle changing application load without manually scaling Deployments.
* Improve resource efficiency by running only the required number of replicas.

### Key points to review / make strong

* HPA purpose
* CPU utilization vs CPU request
* `minReplicas`
* `maxReplicas`
* `targetCPUUtilizationPercentage`
* Metrics Server
* Scale-up vs scale-down
* HPA vs Cluster Autoscaler
* Why HPA changes **Pod count**, not node count

### How it is implemented in our POC

Each microservice has its own HPA:

```text
devops-poc
│
├── user-service
│   ├── Deployment
│   └── HPA
│
├── product-service
│   ├── Deployment
│   └── HPA
│
└── order-service
    ├── Deployment
    └── HPA
```

Our baseline is:

```text
Current replicas: 2
Target CPU:       70%
Maximum replicas: 3
```

So conceptually:

```text
          CPU usage
             │
             ▼
        Metrics Server
             │
             ▼
            HPA
          /     \
       Scale ↑  Scale ↓
          │       │
          ▼       ▼
       Pods      Pods
```

### Core logic

HPA continuously compares the **actual resource utilization** against the configured target.

Example:

```text
Desired:
CPU target = 70%

Actual:
CPU usage becomes high
        ↓
HPA detects utilization
        ↓
Increase replicas
        ↓
2 Pods → 3 Pods
```

When load decreases:

```text
CPU usage decreases
        ↓
HPA evaluates utilization
        ↓
Scale down when appropriate
        ↓
3 Pods → 2 Pods
```

A crucial point:

> HPA does not create new Kubernetes nodes.

It changes the number of **Pod replicas**.

```text
HPA
 ↓
More Pods
 ↓
Existing cluster capacity
```

If there isn't enough node capacity, that's where **Cluster Autoscaler** becomes a separate concern.

### Important factor to remember

One of the most important interview-level concepts:

**HPA works against resource requests.**

For example, if:

```text
CPU request = 100m
CPU usage   = 70m
```

then utilization is approximately:

```text
70 / 100 = 70%
```

So simply saying *"CPU is 70%"* is incomplete. HPA's CPU utilization is evaluated relative to the configured **CPU request**.

Also remember the distinction:

```text
Deployment
   ↓
Defines desired replicas

HPA
   ↓
Dynamically changes replica count

Metrics Server
   ↓
Provides resource metrics
```

### Our POC mental model

```text
User Traffic
     ↓
Ingress
     ↓
Service
     ↓
Pods
     ↑
     │
    HPA
     ↑
Metrics Server
```

So M7 adds **automatic horizontal scaling** on top of the Kubernetes foundation we created in M4.
