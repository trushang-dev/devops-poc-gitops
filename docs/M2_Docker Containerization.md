# M2 — Docker Containerization

### What is it used for

* Package each microservice with its **application code + dependencies + runtime**.
* Make the application run consistently across development, CI, and Kubernetes.
* Provide a deployable artifact: **Docker image**.

### Key points to review / make strong

* Docker image vs container
* Dockerfile
* `FROM`, `WORKDIR`, `COPY`, `RUN`, `EXPOSE`, `CMD`
* Image layers and caching
* `.dockerignore`
* Port mapping
* Environment variables
* Image tagging
* Registry vs local image
* Why containers are useful in Kubernetes

### How it is implemented in our POC

Our application contains three Node.js microservices:

```text
devops-poc-app
│
├── user-service
│   └── Dockerfile
│
├── product-service
│   └── Dockerfile
│
└── order-service
    └── Dockerfile
```

Each service is independently containerized.

The resulting images follow our Docker Hub naming convention:

```text
trushangdev/devops-poc-user-service:<git-sha>
trushangdev/devops-poc-product-service:<git-sha>
trushangdev/devops-poc-order-service:<git-sha>
```

### Core logic

The basic flow is:

```text
Source Code
    ↓
Dockerfile
    ↓
docker build
    ↓
Docker Image
    ↓
Docker Registry
    ↓
Kubernetes pulls image
    ↓
Container runs inside Pod
```

In our POC, the **Git SHA is used as the image tag**.

Example:

```text
trushangdev/devops-poc-user-service:
9858175aee7f8a9469108bebdc0068b18bb259cd
```

This gives us an important relationship:

```text
Git Commit
    ↓
Docker Image
    ↓
Kubernetes Deployment
```

So we can identify **exactly which application source version is running**.

### Important factor to remember

**Image ≠ Container**

Think of it simply:

```text
Docker Image
   = packaged application/template

Container
   = running instance of that image
```

And in our Kubernetes architecture:

```text
Docker Hub
     ↓
  Docker Image
     ↓
 Kubernetes Pod
     ↓
 Container
```

The Docker image is the **artifact**. Kubernetes is responsible for running and managing containers from that artifact.
