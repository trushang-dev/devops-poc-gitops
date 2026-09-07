# DevOps POC — Security

This document describes the security practices, controls, limitations, and operational considerations implemented in the DevOps POC.

The project is a **local DevOps/GitOps learning environment** designed to demonstrate CI/CD, container security, Kubernetes security concepts, GitOps reconciliation, and observability.

It is **not intended to represent a production-ready security architecture**.

---

## 1. Security Objectives

The security approach for this POC focuses on:

- Secure source-code management
- Container image scanning
- Secret separation from application configuration
- Kubernetes Secret usage
- CI/CD security checks
- Immutable image versioning
- GitOps-controlled deployments
- Least-privilege concepts
- Avoiding credentials in source code
- Detecting configuration drift
- Documenting known security limitations

The goal is to demonstrate the security controls that should exist throughout a modern DevOps delivery pipeline.

---

# 2. Security Architecture

```text
                    Developer
                        |
                        v
                ┌───────────────┐
                │    GitHub     │
                │ Source Code   │
                └───────┬───────┘
                        |
                        v
                ┌───────────────┐
                │ GitHub Actions│
                │               │
                │ Tests         │
                │ Build         │
                │ Trivy Scan    │
                └───────┬───────┘
                        |
                        v
                ┌───────────────┐
                │   Docker Hub  │
                │ Container Img │
                └───────┬───────┘
                        |
                        v
                ┌───────────────┐
                │ GitOps Repo   │
                │ Desired State │
                └───────┬───────┘
                        |
                        v
                ┌───────────────┐
                │    Argo CD    │
                │ Reconciliation│
                └───────┬───────┘
                        |
                        v
                ┌───────────────┐
                │  Kubernetes   │
                │               │
                │ Secrets       │
                │ ConfigMaps    │
                │ Pods          │
                │ Services      │
                └───────────────┘
````

Security is therefore treated as a **pipeline concern**, rather than a single Kubernetes feature.

---

# 3. Source Code Security

Application source code is maintained in Git.

Recommended practices:

* Use Pull Requests for changes.
* Avoid committing credentials.
* Avoid committing API keys.
* Avoid committing private keys.
* Avoid committing production configuration.
* Review changes before merging.
* Keep dependencies updated.
* Use GitHub security features where available.

Sensitive values should never be stored directly in application source code.

---

# 4. Secrets

Kubernetes Secrets are used for values that should be separated from normal application configuration.

Application resources contain:

```text
ConfigMap
    ↓
Non-sensitive configuration

Secret
    ↓
Sensitive configuration
```

Example resources:

```text
kubernetes/
├── user-service/
│   ├── configmap.yaml
│   └── secret.yaml
├── product-service/
│   ├── configmap.yaml
│   └── secret.yaml
└── order-service/
    ├── configmap.yaml
    └── secret.yaml
```

### Important limitation

Kubernetes Secrets are **not automatically equivalent to encrypted secrets management**.

This POC uses Kubernetes Secrets for demonstration purposes.

A production environment should evaluate:

* External Secrets
* Cloud secret managers
* HashiCorp Vault
* KMS-backed secret management
* Encryption at rest
* RBAC restrictions
* Secret rotation

---

# 5. Secret Handling Rules

Never commit the following to Git:

```text
.env
.env.*
*.pem
*.key
*.p12
*.pfx
credentials.json
service-account.json
```

Do not place real credentials inside:

* Dockerfiles
* Kubernetes manifests
* Helm values
* GitHub Actions workflows
* application source code
* documentation
* shell scripts

Use placeholders for examples.

Example:

```yaml
DATABASE_PASSWORD: <REDACTED>
```

rather than:

```yaml
DATABASE_PASSWORD: real-production-password
```

---

# 6. Container Security

Container images are built for the application services:

```text
user-service
product-service
order-service
```

Images are scanned using **Trivy** as part of the CI pipeline.

The purpose of scanning is to identify known vulnerabilities in:

* OS packages
* application dependencies
* container images
* other detectable components

The pipeline should prevent known high-risk vulnerabilities from being promoted according to the configured CI policy.

---

# 7. Image Versioning

The project uses versioned container images rather than relying only on:

```text
latest
```

Example:

```text
trushangdev/devops-poc-user-service:v1.0.0
```

A release can therefore be associated with a specific application version.

This provides:

* Traceability
* Repeatability
* Easier rollback
* Clear deployment history
* Better auditability

---

# 8. Immutable Deployment References

The GitOps repository records the image version that Kubernetes should deploy.

Conceptually:

```text
Git commit
     |
     v
Image tag
     |
     v
Helm values
     |
     v
Kubernetes Deployment
```

A deployment can therefore be traced from:

```text
Application commit
        ↓
CI build
        ↓
Container image
        ↓
GitOps commit
        ↓
Argo CD sync
        ↓
Kubernetes Deployment
```

This provides an auditable deployment chain.

---

# 9. CI/CD Security

The CI pipeline performs multiple validation stages before an image is promoted.

Conceptually:

```text
Pull Request
     |
     v
Tests
     |
     v
Build
     |
     v
Security Scan
     |
     v
Image Publish
     |
     v
GitOps Update
```

Security scanning occurs before the image is published as part of the release workflow.

---

# 10. GitOps Security Model

Argo CD is responsible for reconciling the Kubernetes cluster with the GitOps repository.

```text
GitOps Repository
       |
       | Desired State
       v
     Argo CD
       |
       | Reconcile
       v
   Kubernetes
```

This provides an important security and operational property:

> The cluster's intended state is version-controlled.

Manual changes to Kubernetes resources can therefore be detected as drift.

---

# 11. Drift Detection and Self-Healing

The project demonstrates Argo CD drift detection.

Example:

```bash
kubectl set env deployment/user-service \
  DEPLOY_NOTE=manual-drift-test \
  -n devops-poc
```

This intentionally modifies the live Kubernetes state.

The desired state remains defined by Git.

The reconciliation process is:

```text
Git Desired State
       |
       | differs
       v
Kubernetes Live State
       |
       v
Argo CD detects drift
       |
       v
Self-heal
       |
       v
Kubernetes returns to Git state
```

This helps prevent unauthorized or accidental configuration changes from becoming persistent.

---

# 12. Kubernetes Security Controls

The POC demonstrates several Kubernetes security-related concepts.

### Namespaces

Application workloads are isolated in:

```text
devops-poc
```

Monitoring components use:

```text
devops-monitoring
```

Argo CD uses:

```text
argocd
```

Namespaces provide logical separation between workloads.

---

## 12.1 Resource Management

Deployments define resource requests and limits where configured.

This helps Kubernetes make scheduling decisions and prevents uncontrolled resource consumption.

Check configured resources:

```bash
kubectl describe deployment user-service -n devops-poc
```

---

## 12.2 Health Probes

The application uses:

```text
Readiness Probe
Liveness Probe
```

Readiness protects traffic flow:

```text
Not Ready
    ↓
Removed from Service endpoints
```

Liveness supports automatic recovery:

```text
Liveness failure
    ↓
Container restart
```

These controls improve application resilience and reduce the impact of unhealthy containers.

---

# 13. Kubernetes RBAC

Kubernetes RBAC controls which identities can perform which actions.

Inspect available RBAC resources:

```bash
kubectl get roles -A
kubectl get rolebindings -A
kubectl get clusterroles
kubectl get clusterrolebindings
```

### POC limitation

This project does not attempt to implement a complete enterprise RBAC model.

A production implementation should explicitly define:

* ServiceAccounts
* Roles
* RoleBindings
* ClusterRoles where required
* Least-privilege permissions
* Separate identities for workloads and automation

---

# 14. Network Security

Kubernetes Services provide internal service discovery and network abstraction.

The application is separated into individual services:

```text
user-service
product-service
order-service
```

Ingress provides the external HTTP routing layer.

```text
Client
  |
  v
Ingress
  |
  +--> user-service
  |
  +--> product-service
  |
  +--> order-service
```

### POC limitation

The project does not currently implement a complete Kubernetes NetworkPolicy model.

A production environment should consider explicit NetworkPolicies to restrict:

```text
Who can talk to whom?
```

For example:

```text
Ingress
   ↓
Application Services
   ↓
Database
```

while preventing unnecessary lateral communication.

---

# 15. Helm Security Considerations

Helm is used to package the application Kubernetes resources.

Main chart:

```text
helm/microservices/
```

Configuration should be reviewed carefully because Helm values can influence:

* container images
* environment variables
* resources
* replica counts
* probes
* Services
* Ingress
* other Kubernetes configuration

Sensitive credentials should not be stored directly in ordinary Helm values files.

---

# 16. GitHub Actions Security

GitHub Actions is responsible for CI/CD automation.

Security considerations include:

* Store credentials as GitHub Actions Secrets.
* Do not hard-code tokens.
* Avoid printing secrets in workflow logs.
* Restrict workflow permissions where possible.
* Review third-party Actions before use.
* Pin Actions appropriately for stronger supply-chain control.
* Protect release branches.
* Require Pull Request review where appropriate.

---

# 17. Docker Registry Security

Container images are published to Docker Hub.

Credentials should be supplied through CI/CD secrets rather than stored in workflow files.

Conceptually:

```text
GitHub Actions
      |
      | Registry credentials
      | from secure secret storage
      v
Docker Hub
```

The GitOps repository should contain image references, not registry passwords.

---

# 18. Dependency Security

Node.js services use npm dependencies.

Dependency management should include:

```bash
npm audit
```

and regular dependency updates.

Package lock files are committed:

```text
package-lock.json
```

This improves reproducibility of dependency installation.

Container scanning provides an additional security layer beyond application dependency checks.

---

# 19. Security Verification

Useful checks include:

### Check Kubernetes Secrets

```bash
kubectl get secrets -n devops-poc
```

Do not expose Secret values unnecessarily.

---

### Check Container Images

```bash
kubectl get deployments -n devops-poc \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image'
```

---

### Check Running Pods

```bash
kubectl get pods -n devops-poc
```

---

### Check ServiceAccounts

```bash
kubectl get serviceaccounts -n devops-poc
```

---

### Check RBAC

```bash
kubectl get roles,rolebindings -n devops-poc
```

---

### Check Ingress

```bash
kubectl get ingress -n devops-poc
```

---

# 20. Public Repository Security Checklist

Before making this project public:

* [ ] No passwords committed
* [ ] No API keys committed
* [ ] No access tokens committed
* [ ] No private keys committed
* [ ] No production credentials committed
* [ ] `.env` files excluded
* [ ] Docker credentials excluded
* [ ] GitHub Actions secrets are referenced securely
* [ ] Git history reviewed for accidental secrets
* [ ] Container images scanned
* [ ] README does not contain sensitive environment information
* [ ] Example configuration uses placeholders
* [ ] Security policy reviewed

A repository being public means the **entire Git history should be treated as potentially visible**.

Removing a secret from the latest commit does not make it safe if that secret remains in previous commits.

---

# 21. Known Security Limitations

This POC intentionally has limitations because it is a local learning environment.

Current limitations include:

| Area               | Current POC                           | Production Consideration               |
| ------------------ | ------------------------------------- | -------------------------------------- |
| Kubernetes Secrets | Basic Secret resources                | External secret manager                |
| Secret encryption  | Not implemented as production control | KMS / encryption at rest               |
| NetworkPolicy      | Not implemented                       | Explicit network segmentation          |
| RBAC               | Basic/default environment             | Least-privilege RBAC                   |
| TLS                | Local/demo configuration              | TLS certificates + automated renewal   |
| Registry           | Docker Hub                            | Private registry / enterprise controls |
| Image signing      | Not implemented                       | Cosign / Sigstore                      |
| Admission control  | Not implemented                       | Kyverno / OPA Gatekeeper               |
| Runtime security   | Not implemented                       | Runtime detection and policy           |
| Supply chain       | Basic CI scanning                     | SBOM + signing + provenance            |
| Cluster            | Minikube                              | Managed/production Kubernetes          |
| Secrets rotation   | Manual                                | Automated rotation                     |

These limitations are intentional and documented rather than hidden.

---

# 22. Security Philosophy

The project follows a layered approach:

```text
                 Security Layers

              ┌──────────────────┐
              │ GitHub Security  │
              └────────┬─────────┘
                       ↓
              ┌──────────────────┐
              │ CI Validation    │
              │ Tests + Trivy    │
              └────────┬─────────┘
                       ↓
              ┌──────────────────┐
              │ Container Image  │
              │ Versioning       │
              └────────┬─────────┘
                       ↓
              ┌──────────────────┐
              │ GitOps           │
              │ Argo CD          │
              └────────┬─────────┘
                       ↓
              ┌──────────────────┐
              │ Kubernetes       │
              │ Secrets / Probes │
              │ Resources        │
              └────────┬─────────┘
                       ↓
              ┌──────────────────┐
              │ Observability    │
              │ Prometheus       │
              │ Grafana / OTel   │
              └──────────────────┘
```

No individual control provides complete security.

The objective is to create multiple layers that reduce the probability and impact of mistakes, vulnerabilities, and unauthorized changes.

---

# 23. Related Documentation

| Document                                                    | Purpose                         |
| ----------------------------------------------------------- | ------------------------------- |
| `README.md`                                                 | Project overview                |
| `ARCHITECTURE.md`                                           | Complete system architecture    |
| `PROJECT_JOURNEY.md`                                        | Project evolution               |
| `RUNBOOK.md`                                                | Operational procedures          |
| `TROUBLESHOOTING.md`                                        | Failure investigation           |
| `OBSERVABILITY.md`                                          | Monitoring and telemetry        |
| `M9_GitOps Repository & Deployment Configuration.md`        | GitOps implementation           |
| `M11_Docker Hub & Immutable Image Versioning.md`            | Image publishing and versioning |
| `M12_Observability: Prometheus, Grafana & OpenTelemetry.md` | Observability implementation    |
| `M13_Argo CD & GitOps Reconciliation.md`                    | Argo CD reconciliation          |
| `PROJECT_JOURNEY.md` (Phase 8)                              | Failure testing                 |
| `PROJECT_JOURNEY.md` (Phase 9)                              | End-to-end DevSecOps validation |

---

## Disclaimer

This project is a **DevOps/GitOps learning POC**.

The security controls demonstrated here are intended to illustrate concepts and operational practices. They should not be interpreted as a complete production security architecture.

Production environments require additional controls based on their infrastructure, threat model, compliance requirements, identity architecture, data sensitivity, and organizational security policies.
