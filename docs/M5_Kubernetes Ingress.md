# M5 — Kubernetes Ingress

### What is it used for

* Expose multiple Kubernetes services through a **single external entry point**.
* Route incoming HTTP requests to the correct microservice.
* Provide path-based routing for our three services.

### Key points to review / make strong

* Ingress vs Service
* Ingress Controller
* NGINX Ingress Controller
* Host-based vs path-based routing
* `IngressClass`
* Path matching
* Backend Service
* Why Ingress is needed instead of exposing every Service externally
* Difference between **Ingress resource** and **Ingress Controller**

### How it is implemented in our POC

Our Minikube cluster uses the **NGINX Ingress Controller**.

Traffic enters through:

```text
Client
   ↓
NGINX Ingress Controller
   ↓
Kubernetes Service
   ↓
Pod
```

Our application uses path-based routing:

```text
/api/users/*      → user-service
/api/products/*   → product-service
/api/orders/*     → order-service
```

Health-check paths are also explicitly routed:

```text
/api/users/health
/api/products/health
/api/orders/health
```

### Core logic

The Ingress doesn't run the application.

It simply decides:

> **"Based on this incoming request, which Kubernetes Service should receive it?"**

For example:

```text
GET /api/users/health
        ↓
NGINX Ingress
        ↓
user-service
        ↓
user-service Pod
        ↓
Node.js application
        ↓
200 OK
```

Similarly:

```text
/api/products/*
        ↓
product-service

/api/orders/*
        ↓
order-service
```

### Important factor to remember

There are **three different things** to keep separate:

```text
Ingress
   ↓
Routing rules

Ingress Controller
   ↓
Actual component that receives/processes traffic

Service
   ↓
Stable endpoint for Pods
```

So our request flow is:

```text
Internet / Browser
       ↓
Minikube IP
       ↓
NGINX Ingress Controller
       ↓
Ingress rules
       ↓
Service
       ↓
Pod
       ↓
Container
```

And this becomes especially important later when we add **Grafana, Prometheus and Argo CD** behind the same NGINX entry point.
