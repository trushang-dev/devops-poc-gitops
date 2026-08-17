# M0 — Project Foundation

**Filename:** `M0_Project_Foundation.md`

## 1. What is it used for?

M0 establishes the **base structure and working foundation** of the DevOps POC.

The goal is to make sure the project has:

* Proper repositories
* Git structure
* Initial documentation
* Development workflow
* A clear place for application code and infrastructure/GitOps configuration

Think of M0 as:

> **“Before building DevOps automation, establish a clean and manageable project foundation.”**

---

## 2. What did we implement in our POC?

Your POC is separated into two major repositories:

```text
devops-poc-app
        │
        └── Application source code
            ├── user-service
            ├── product-service
            └── order-service


devops-poc-gitops
        │
        └── Deployment / infrastructure configuration
            ├── Helm
            ├── Kubernetes
            ├── Monitoring
            └── Argo CD
```

This separation becomes **very important later** because your architecture follows GitOps.

### Application repository

Contains the actual Node.js microservices and their development lifecycle.

```text
Developer
   ↓
devops-poc-app
   ↓
CI
```

### GitOps repository

Contains the desired Kubernetes state.

```text
devops-poc-gitops
        ↓
     Argo CD
        ↓
   Kubernetes
```

So even at M0, the project structure is preparing us for the later:

```text
CI ≠ CD
```

---

## 3. Core Logic

### ① Separate application code from deployment configuration

This is one of the most important architectural decisions.

```text
Application Repo
      │
      │ builds
      ↓
 Docker Image
```

while:

```text
GitOps Repo
      │
      │ defines desired state
      ↓
 Kubernetes
```

This separation allows CI and CD to evolve independently.

---

### ② Git is the foundation

Git tracks changes to:

* Application code
* Docker configuration
* Kubernetes configuration
* Helm configuration
* CI/CD configuration
* Documentation

Later, Git becomes more than source-code version control.

With GitOps:

> **Git becomes the source of truth for the desired Kubernetes state.**

---

### ③ Repository structure matters

The project deliberately separates responsibilities.

```text
devops-poc-app
       │
       ├── source code
       ├── tests
       ├── Dockerfiles
       └── CI workflow


devops-poc-gitops
       │
       ├── Helm
       ├── Kubernetes
       ├── Monitoring
       └── Argo CD
```

This makes the later deployment flow much easier to reason about.

---

## 4. Key Points I Must Make Strong

### 🔥 Must Know

* Why application and GitOps repositories are separated
* Git as the source of truth
* Difference between **application repository** and **GitOps repository**
* Why infrastructure/deployment configuration should be version controlled
* Basic Git branching/workflow used by the project
* Repository ownership/responsibility

### ⭐ Should Know

* Why Git history is important for deployment troubleshooting
* Why infrastructure configuration should be reviewable through Git
* Why separating application and deployment concerns helps CI/CD

### 💡 Good to Know

* How this structure can scale to multiple environments
* How the GitOps repository could later contain `dev`, `staging`, and `production` configuration

---

## 5. Important Factors / Gotchas

### ⚠️ Don't confuse the two repositories

A common mistake is thinking:

> "The GitOps repository contains the application."

It doesn't.

It contains the **desired deployment state/configuration** for the application.

---

### ⚠️ GitOps means Git is authoritative

Later in the project, you have:

```text
GitOps Repository
        ↓
      Argo CD
        ↓
    Kubernetes
```

Therefore, manually changing Kubernetes resources can create **drift**.

This becomes particularly important when we review M13.

---

### ⚠️ CI and CD have different responsibilities

Your eventual architecture intentionally separates:

```text
CI
↓
Test
Build
Scan
Publish image
Update GitOps
```

from:

```text
CD
↓
Argo CD
↓
Kubernetes reconciliation
```

That distinction starts with the foundation established in M0.

---

## 6. How It Connects to the Overall Architecture

M0 is the starting point:

```text
                M0 Foundation
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
   Application Repo        GitOps Repo
          │                     │
          ↓                     ↓
       M1/M3...              M8...
          │                     │
          ↓                     ↓
       CI/CD              Kubernetes
                                │
                                ↓
                             Argo CD
```

Eventually this becomes:

```text
Developer
   ↓
Application Repo
   ↓
GitHub Actions
   ↓
Docker Image
   ↓
GitOps Repo
   ↓
Argo CD
   ↓
Kubernetes
```

---

## 7. Hands-on Commands / Verification

### Check repository status

```bash
git status
```

### Check current branch

```bash
git branch
```

### Check remote repository

```bash
git remote -v
```

### Review recent history

```bash
git log --oneline -10
```

### See branches

```bash
git branch -a
```

### See tracked files

```bash
git ls-files
```

### Compare working tree

```bash
git diff
```

For your two repositories, the basic verification is:

```bash
cd devops-poc-app
git status
git remote -v
git log --oneline -5
```

and:

```bash
cd devops-poc-gitops
git status
git remote -v
git log --oneline -5
```

---

# 8. Memory Cheat Sheet

**Remember this:**

* M0 = **Project Foundation**
* Two repositories → **App + GitOps**
* App repo = **source code**
* GitOps repo = **desired deployment state**
* Git tracks everything
* Git provides history and traceability
* Application lifecycle and deployment lifecycle are separated
* GitOps repo later becomes the source of truth for Kubernetes
* CI builds/publishes
* CD reconciles/deploys
* **M0 sets the foundation for everything after it**

---

# 9. One-Line Mental Model

```text
App Code → Application Repo

Deployment State → GitOps Repo
```

And the bigger picture:

```text
Application Repo → CI → Image → GitOps Repo → Argo CD → Kubernetes
```

### ⚠️ M0 Status

**✅ Completed**

No major unverified area needs to be added to the handbook based on the milestone information we've established.
