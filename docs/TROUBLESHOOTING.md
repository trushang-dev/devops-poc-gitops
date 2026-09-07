# DevOps POC — Troubleshooting Guide

This document provides a practical troubleshooting guide for the DevOps POC.

It focuses on common Kubernetes, Docker, Helm, GitOps, CI/CD, observability, and local-environment issues encountered while building and validating the project.

The troubleshooting approach is:

```text
Symptom
   ↓
Check current state
   ↓
Collect evidence
   ↓
Identify responsible layer
   ↓
Find root cause
   ↓
Apply minimal fix
   ↓
Verify recovery
````

---

# 1. Troubleshooting Philosophy

When something fails, do not immediately restart or recreate everything.

Start by determining **which layer is actually failing**.

```text
Application
    ↓
Container
    ↓
Pod
    ↓
Deployment / ReplicaSet
    ↓
Service
    ↓
Ingress
    ↓
Kubernetes
    ↓
Argo CD
    ↓
GitOps Repository
    ↓
CI/CD
    ↓
Container Registry
```

A failure in one layer does not necessarily mean another layer is broken.

For example:

* A Pod restarting does not mean Argo CD is broken.
* A Service returning no endpoints does not mean the Deployment is broken.
* An Argo CD `OutOfSync` state does not necessarily mean Kubernetes is unhealthy.
* A failed Ingress request does not necessarily mean the application is down.

---

# 2. First Response Checklist

When something appears broken, start with:

```bash
minikube status
```

Then:

```bash
kubectl get nodes
```

Check application Pods:

```bash
kubectl get pods -n devops-poc
```

Check Deployments:

```bash
kubectl get deployments -n devops-poc
```

Check Services:

```bash
kubectl get svc -n devops-poc
```

Check endpoints:

```bash
kubectl get endpoints -n devops-poc
```

Check Ingress:

```bash
kubectl get ingress -n devops-poc
```

Check Argo CD:

```bash
kubectl get application devops-poc -n argocd
```

Check recent events:

```bash
kubectl get events -n devops-poc --sort-by=.lastTimestamp
```

This gives a quick view of the complete application stack.

---

# 3. Minikube Is Not Running

## Symptom

Commands such as:

```bash
kubectl get pods
```

fail or cannot connect to the Kubernetes API.

## Check

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

## Recovery

Start Minikube:

```bash
minikube start --driver=docker
```

Verify:

```bash
kubectl get nodes
```

---

# 4. Docker / Minikube Cannot Start

## Symptom

Minikube reports errors connecting to Docker or the Docker API.

Example symptoms may include:

```text
failed to connect to docker API
```

## Investigation

Check Docker:

```bash
docker info
```

Check Docker service:

```bash
systemctl status docker
```

Check disk space:

```bash
df -h
```

Check Docker disk usage:

```bash
docker system df
```

## Important

The POC uses multiple container image versions for:

* deployment testing
* rollback testing
* failure testing

Therefore, do not immediately run:

```bash
docker system prune -a
```

without checking which images and containers are still required.

---

# 5. Disk Space Is Almost Full

## Symptom

Docker or Minikube becomes unstable.

Pods may fail unexpectedly or the Docker environment may stop behaving normally.

## Check

```bash
df -h
```

Check Docker:

```bash
docker system df
```

Check Minikube:

```bash
minikube status
```

## Investigation

Identify large Docker resources:

```bash
docker images
docker ps -a
docker volume ls
```

Remove only resources known to be unnecessary.

## Important

During the POC, Docker disk usage reached very high levels during repeated image builds and Kubernetes experiments.

This is a local-environment resource problem and should not automatically be interpreted as an application or Kubernetes configuration problem.

---

# 6. Pod Is in CrashLoopBackOff

## Symptom

```bash
kubectl get pods -n devops-poc
```

shows:

```text
CrashLoopBackOff
```

## Check Pod details

```bash
kubectl describe pod <pod-name> -n devops-poc
```

Check logs:

```bash
kubectl logs <pod-name> -n devops-poc
```

Check previous container logs:

```bash
kubectl logs <pod-name> -n devops-poc --previous
```

Check restart count:

```bash
kubectl get pods -n devops-poc
```

## Check events

```bash
kubectl get events -n devops-poc --sort-by=.lastTimestamp
```

## Common causes

Possible causes include:

* Application process exits
* Incorrect environment configuration
* Missing Secret/ConfigMap
* Failed startup dependency
* Liveness probe failure
* Container image problem
* Resource constraints

Do not assume the cause until logs and events are inspected.

---

# 7. Pod Is Running but Not Ready

## Symptom

A Pod shows:

```text
Running
```

but:

```text
READY
```

is not `1/1`.

Example:

```text
1/1
0/1
```

## Check

```bash
kubectl get pods -n devops-poc
```

Inspect:

```bash
kubectl describe pod <pod-name> -n devops-poc
```

Look for:

```text
Readiness probe failed
```

## Why this matters

A Pod can be running while still being unavailable to application traffic.

```text
Container Running
       |
       X
       |
Readiness = false
       |
       ↓
Not included in Service endpoints
```

Check endpoints:

```bash
kubectl get endpoints -n devops-poc
```

---

# 8. Liveness Probe Is Restarting the Container

## Symptom

Pod restart count continually increases.

Check:

```bash
kubectl get pods -n devops-poc
```

Inspect:

```bash
kubectl describe pod <pod-name> -n devops-poc
```

Look for:

```text
Liveness probe failed
```

## Understand the difference

### Readiness

Controls whether the Pod receives traffic.

```text
Readiness failure
      ↓
Pod removed from endpoints
```

### Liveness

Determines whether the container should be restarted.

```text
Liveness failure
      ↓
Container restart
```

Do not treat these two probes as interchangeable.

---

# 9. Pod Was Deleted but Came Back

## Symptom

A Pod was manually deleted:

```bash
kubectl delete pod <pod-name> -n devops-poc
```

but another Pod appears automatically.

## Expected behavior

This is normal.

The Deployment manages a ReplicaSet, which maintains the desired number of Pods.

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

If a Pod disappears:

```text
Pod deleted
    ↓
ReplicaSet detects fewer replicas
    ↓
New Pod created
```

## Important

This recovery is performed by **Kubernetes**, not Argo CD.

---

# 10. Deployment Rollout Is Stuck

## Check status

```bash
kubectl rollout status deployment/user-service -n devops-poc
```

Check Deployment:

```bash
kubectl describe deployment user-service -n devops-poc
```

Check ReplicaSets:

```bash
kubectl get replicasets -n devops-poc
```

Check Pods:

```bash
kubectl get pods -n devops-poc
```

Check events:

```bash
kubectl get events -n devops-poc --sort-by=.lastTimestamp
```

## Possible causes

* New Pods cannot start
* Image cannot be pulled
* Readiness probe failing
* Resource constraints
* Application startup failure
* Invalid Deployment configuration

---

# 11. ImagePullBackOff

## Symptom

Pod shows:

```text
ImagePullBackOff
```

or:

```text
ErrImagePull
```

## Check

```bash
kubectl describe pod <pod-name> -n devops-poc
```

Look at the image:

```bash
kubectl get deployment user-service -n devops-poc \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Verify the image exists in the registry.

Check the image reference for:

* repository name
* image name
* tag

Example:

```text
trushangdev/devops-poc-user-service:v1.0.0
```

---

# 12. Service Has No Endpoints

## Symptom

Service exists:

```bash
kubectl get svc -n devops-poc
```

but has no backend endpoints.

## Check

```bash
kubectl get endpoints -n devops-poc
```

Inspect the Service:

```bash
kubectl describe svc user-service -n devops-poc
```

Check Pod labels:

```bash
kubectl get pods -n devops-poc --show-labels
```

Check Service selector:

```bash
kubectl get svc user-service -n devops-poc -o yaml
```

## Common cause

The Service selector does not match the Pod labels.

Conceptually:

```text
Service Selector
      |
      | must match
      ↓
Pod Labels
      |
      ↓
Endpoints
```

If the Pod is not Ready, it may also be absent from the usable endpoints.

---

# 13. Ingress Is Not Routing Correctly

## Symptom

The application works internally but an external request fails.

## First check

```bash
kubectl get ingress -n devops-poc
```

Inspect:

```bash
kubectl describe ingress <ingress-name> -n devops-poc
```

Check ingress controller:

```bash
kubectl get pods -n ingress-nginx
```

Check Services:

```bash
kubectl get svc -n devops-poc
```

Check endpoints:

```bash
kubectl get endpoints -n devops-poc
```

## Debugging principle

Test from the inside out:

```text
Pod
 ↓
Service
 ↓
Ingress
 ↓
Client
```

If the application does not work inside the Pod, debugging Ingress first is unnecessary.

---

# 14. Ingress Path / Regex Issue

The project encountered an Ingress routing issue involving path/regex behavior.

When a route does not match as expected:

1. Inspect the actual Ingress resource.
2. Inspect annotations.
3. Check path definitions.
4. Check path ordering.
5. Test the backend directly.
6. Test the Service.
7. Test through Ingress.

Useful commands:

```bash
kubectl describe ingress <ingress-name> -n devops-poc
```

Test the application directly inside the Pod:

```bash
kubectl exec deployment/user-service -n devops-poc -- \
  wget -qO- http://localhost:3001/version
```

This provides a way to distinguish:

```text
Application problem
```

from:

```text
Ingress routing problem
```

---

# 15. HPA Is Not Scaling

## Check HPA

```bash
kubectl get hpa -n devops-poc
```

Detailed information:

```bash
kubectl describe hpa <hpa-name> -n devops-poc
```

Check current Pods:

```bash
kubectl get pods -n devops-poc
```

Check metrics:

```bash
kubectl top pods -n devops-poc
```

Check metrics-server:

```bash
kubectl get pods -n kube-system | grep metrics
```

## Important

If metrics-server is unhealthy, HPA may not receive the metrics required for scaling.

Do not debug HPA configuration alone without checking the metrics source.

---

# 16. Helm and HPA Replica Ownership

## Problem

A Deployment may have a replica count defined in Helm while an HPA is also controlling the replica count.

This can create confusing behavior.

Conceptually:

```text
Helm
  |
  | desired base configuration
  v
Deployment <---- HPA
                 |
                 | runtime scaling
                 v
              replicas
```

## Investigation

Check Deployment:

```bash
kubectl get deployment user-service -n devops-poc -o yaml
```

Check HPA:

```bash
kubectl get hpa user-service -n devops-poc -o yaml
```

Check Helm values:

```bash
grep -n "replica" helm/microservices/values.yaml
```

The configuration should clearly define which component owns runtime replica scaling.

---

# 17. Argo CD Shows OutOfSync

## Check

```bash
kubectl get application devops-poc -n argocd
```

Get detailed information:

```bash
kubectl describe application devops-poc -n argocd
```

## Understand the state

```text
Synced
   =
Live state matches Git

OutOfSync
   =
Live state differs from Git
```

## Possible causes

* Manual Kubernetes change
* GitOps repository change
* Helm-rendered difference
* Resource modification
* Argo CD configuration difference

Check application resources:

```bash
kubectl get application devops-poc -n argocd \
  -o jsonpath='{range .status.resources[*]}{.kind}{"\t"}{.name}{"\t"}{.status}{"\n"}{end}'
```

---

# 18. Argo CD Self-Healing

The project intentionally tested Kubernetes drift.

Example:

```bash
kubectl set env deployment/user-service \
  DEPLOY_NOTE=manual-drift-test \
  -n devops-poc
```

Argo CD detected the difference and reconciled the resource back to the Git-defined state.

Important distinction:

```text
Manual Kubernetes change
        |
        v
Kubernetes changes immediately
        |
        v
Argo CD detects drift
        |
        v
Argo CD self-heals
```

Kubernetes itself does not compare the live state against Git.

---

# 19. Argo CD Is Synced but Application Is Broken

This is possible.

```text
Argo CD
Synced
Healthy
```

does not mean the application is necessarily behaving correctly from an end-user perspective.

Argo CD primarily evaluates the declared Kubernetes resources and configured health checks.

Continue troubleshooting:

```bash
kubectl get pods -n devops-poc
kubectl get svc -n devops-poc
kubectl get endpoints -n devops-poc
kubectl logs deployment/user-service -n devops-poc
```

Then test the application endpoint directly.

---

# 20. GitOps Rollback

If the GitOps repository contains a bad deployment state:

```bash
git log --oneline
```

Identify the problematic commit.

Revert it:

```bash
git revert <commit>
```

Push:

```bash
git push origin main
```

Argo CD should detect the new Git state and reconcile Kubernetes.

Verify:

```bash
kubectl get application devops-poc -n argocd
```

Then:

```bash
kubectl get deployments -n devops-poc
```

---

# 21. Kubernetes Rollback

For a Deployment-level rollback:

```bash
kubectl rollout history deployment/user-service -n devops-poc
```

Rollback:

```bash
kubectl rollout undo deployment/user-service -n devops-poc
```

Verify:

```bash
kubectl rollout status deployment/user-service -n devops-poc
```

Check image:

```bash
kubectl get deployment user-service -n devops-poc \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### Important distinction

```text
Kubernetes Rollback
        |
        ↓
Changes live cluster state

GitOps Rollback
        |
        ↓
Changes Git desired state
        |
        ↓
Argo CD reconciles cluster
```

---

# 22. Grafana Is Not Showing a Dashboard

## Check Grafana Pod

```bash
kubectl get pods -n devops-monitoring | grep grafana
```

Check dashboard ConfigMaps:

```bash
kubectl get configmaps -n devops-monitoring \
  -l grafana_dashboard=1
```

Inspect a ConfigMap:

```bash
kubectl describe configmap <dashboard-configmap> \
  -n devops-monitoring
```

Check Grafana logs:

```bash
kubectl logs <grafana-pod> -n devops-monitoring
```

If the dashboard sidecar is present, inspect its logs as well:

```bash
kubectl logs <grafana-pod> \
  -n devops-monitoring \
  -c grafana-sc-dashboard
```

The dashboard provisioning flow is:

```text
Kubernetes ConfigMap
        |
        | grafana_dashboard=1
        v
Grafana Sidecar
        |
        v
Dashboard JSON
        |
        v
Grafana
```

---

# 23. Prometheus Is Not Scraping Application Metrics

## Check Prometheus

```bash
kubectl get pods -n devops-monitoring | grep prometheus
```

Application metrics are scraped via a per-service `PodMonitor`, not a `ServiceMonitor` — Prometheus discovers each Pod directly rather than going through the Service's ClusterIP (see `OBSERVABILITY.md` for why). Check PodMonitors:

```bash
kubectl get podmonitors -n devops-poc
```

Inspect:

```bash
kubectl describe podmonitor <name> -n devops-poc
```

Check application Pods (a PodMonitor targets Pods directly, not the Service):

```bash
kubectl get pods -n devops-poc -o wide
```

Check endpoints:

```bash
kubectl get endpoints -n devops-poc
```

The debugging path is:

```text
Application
   ↓
/metrics endpoint
   ↓
Pod (discovered directly)
   ↓
PodMonitor
   ↓
Prometheus
   ↓
Grafana
```

Find the first broken layer.

---

# 24. OpenTelemetry Metrics Appear Duplicated

The project encountered an observability configuration issue where metrics could be collected through overlapping telemetry paths.

When metrics appear duplicated:

1. Identify all scrape configurations.
2. Check PodMonitors (application metrics) and ServiceMonitors (e.g. the OTel Collector's own self-telemetry).
3. Check OpenTelemetry Collector configuration — a Collector *receiver* can independently scrape the same target outside of any PodMonitor/ServiceMonitor object, which is exactly what caused this project's real duplicate-metrics bug (see `OBSERVABILITY.md`).
4. Determine whether both Prometheus and OTel are collecting the same metric.
5. Verify the intended telemetry pipeline.

Check Collector configuration:

```bash
kubectl get configmaps -n devops-monitoring
```

Check Collector logs:

```bash
kubectl logs <otel-pod> -n devops-monitoring
```

The key question is:

> **How many independent paths are collecting the same metric?**

---

# 25. Grafana or Monitoring Pod Restarted

Check Pod status:

```bash
kubectl get pods -n devops-monitoring
```

Check restart counts:

```bash
kubectl get pods -n devops-monitoring
```

Inspect the Pod:

```bash
kubectl describe pod <pod-name> -n devops-monitoring
```

Check logs:

```bash
kubectl logs <pod-name> -n devops-monitoring
```

Check previous logs if the container restarted:

```bash
kubectl logs <pod-name> -n devops-monitoring --previous
```

Also check node resources:

```bash
kubectl top nodes
```

and disk usage:

```bash
df -h
```

During the POC, local resource pressure contributed to monitoring instability.

---

# 26. metrics-server Is Unhealthy

Check:

```bash
kubectl get pods -n kube-system | grep metrics
```

Inspect:

```bash
kubectl describe pod <metrics-server-pod> -n kube-system
```

Logs:

```bash
kubectl logs <metrics-server-pod> -n kube-system
```

### Important

A metrics-server issue can affect:

* `kubectl top`
* HPA metrics

but does not automatically mean:

* application Pods are broken
* Argo CD is broken
* Prometheus is broken
* Grafana is broken

Treat it as a separate platform dependency.

---

# 27. Application Works Inside Pod but Not Through Ingress

Test internally:

```bash
kubectl exec deployment/user-service -n devops-poc -- \
  wget -qO- http://localhost:3001/version
```

If this works:

```text
Application
    ✓
```

Continue outward:

```text
Service
   ↓
Ingress
   ↓
Client
```

Check Service:

```bash
kubectl get svc user-service -n devops-poc
```

Check endpoints:

```bash
kubectl get endpoints user-service -n devops-poc
```

Check Ingress:

```bash
kubectl describe ingress <ingress-name> -n devops-poc
```

---

# 28. CI Build Succeeds but Deployment Does Not Update

Follow the deployment chain:

```text
GitHub Actions
      ↓
Docker Image
      ↓
Docker Hub
      ↓
GitOps Commit
      ↓
Argo CD
      ↓
Kubernetes
```

Check each layer.

### 1. GitHub Actions

Verify workflow succeeded.

### 2. Container image

Check the expected image/tag exists.

### 3. GitOps repository

Check:

```bash
git log --oneline -10
```

Confirm the expected image tag was committed.

### 4. Argo CD

```bash
kubectl get application devops-poc -n argocd
```

### 5. Kubernetes

```bash
kubectl get deployments -n devops-poc
```

Check the actual image:

```bash
kubectl get deployment user-service -n devops-poc \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

---

# 29. Deployment Updated but Old Pods Remain

Check Deployment:

```bash
kubectl get deployment user-service -n devops-poc
```

Check ReplicaSets:

```bash
kubectl get replicasets -n devops-poc
```

Check Pods:

```bash
kubectl get pods -n devops-poc
```

Check rollout:

```bash
kubectl rollout status deployment/user-service -n devops-poc
```

If the rollout is still in progress, old Pods may temporarily coexist with new Pods.

This is expected during a Kubernetes `RollingUpdate`.

---

# 30. How to Determine Which Layer Owns the Problem

Use this table:

| Symptom                       | First Component to Check         |
| ----------------------------- | -------------------------------- |
| Pod disappears                | ReplicaSet / Deployment          |
| Pod restarts                  | Container logs / probes          |
| Pod not Ready                 | Readiness probe                  |
| No Service endpoints          | Service selector / Pod readiness |
| Service unreachable           | Service + endpoints              |
| Ingress unreachable           | Ingress + controller             |
| HPA not scaling               | HPA + metrics-server             |
| Argo OutOfSync                | Git vs live state                |
| Argo sync failure             | Application + resource events    |
| Dashboard missing             | ConfigMap + Grafana sidecar      |
| Metrics missing               | PodMonitor + Prometheus          |
| Metrics duplicated            | Prometheus + OTel pipelines      |
| CI succeeds but no deployment | GitOps commit + Argo CD          |
| Minikube unavailable          | Docker + Minikube                |
| Docker unstable               | Disk / Docker resources          |

---

# 31. Evidence Collection

When documenting a failure, collect evidence before making changes.

Useful commands:

```bash
kubectl get pods -A
```

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

```bash
kubectl describe pod <pod-name> -n <namespace>
```

```bash
kubectl logs <pod-name> -n <namespace>
```

```bash
kubectl get deployments -A
```

```bash
kubectl get svc -A
```

```bash
kubectl get endpoints -A
```

```bash
kubectl get application devops-poc -n argocd
```

For local infrastructure:

```bash
minikube status
docker info
df -h
docker system df
```

---

# 32. Recovery Principles

When recovering from a failure:

### 1. Avoid destructive actions first

Do not immediately:

```text
delete cluster
delete all containers
prune all Docker images
reinstall Kubernetes
```

First determine the failure.

### 2. Fix the correct layer

If a Pod is unhealthy, changing Argo CD configuration is unlikely to help.

If Ingress routing is broken, rebuilding the container is unnecessary.

### 3. Preserve evidence

Before deleting a failed resource, inspect:

```bash
kubectl describe
kubectl logs
kubectl get events
```

### 4. Verify after recovery

Recovery is not complete until the expected state is confirmed.

For example:

```text
Pod Running
+
Pod Ready
+
Service endpoints populated
+
Application responds
+
Argo CD Synced
```

---

# 33. Troubleshooting Decision Tree

```text
                 Something is broken
                         |
                         v
                Is Minikube running?
                   /           \
                 NO             YES
                 |               |
          Start Minikube        v
                          Are Pods healthy?
                            /        \
                          NO          YES
                          |            |
                    Check logs        v
                    + events     Are endpoints present?
                                      /       \
                                    NO         YES
                                    |           |
                              Check Service     v
                              + readiness   Does Ingress work?
                                              /       \
                                            NO         YES
                                            |           |
                                      Check Ingress     v
                                      + controller   Check
                                                     application
                                                       |
                                                       v
                                                  Check metrics
                                                  / dashboards
```

---

# 34. Related Documentation

| Document                                                    | Purpose                         |
| ----------------------------------------------------------- | ------------------------------- |
| `README.md`                                                 | Project overview                |
| `ARCHITECTURE.md`                                           | System architecture             |
| `PROJECT_JOURNEY.md`                                        | Project evolution               |
| `RUNBOOK.md`                                                | Normal operational procedures   |
| `OBSERVABILITY.md`                                          | Monitoring and telemetry        |
| `SECURITY.md`                                               | Security practices              |
| `M5_Kubernetes Ingress.md`                                  | Ingress implementation          |
| `M7_Kubernetes HPA.md`                                      | HPA implementation              |
| `M12_Observability: Prometheus, Grafana & OpenTelemetry.md` | Observability implementation    |
| `M13_Argo CD & GitOps Reconciliation.md`                    | GitOps reconciliation           |
| `PROJECT_JOURNEY.md` (Phase 8)                              | Failure testing                 |
| `PROJECT_JOURNEY.md` (Phase 9)                              | End-to-end DevSecOps validation |

---

# 35. Key Takeaways

The most important troubleshooting lessons from this POC are:

1. **Start with evidence, not assumptions.**
2. **Troubleshoot from the inside out.**
3. **Separate Kubernetes problems from Argo CD problems.**
4. **Separate application problems from networking problems.**
5. **Understand controller ownership.**
6. **Use Events + Describe + Logs together.**
7. **Preserve evidence before deleting failed resources.**
8. **Avoid destructive cleanup until resource usage is understood.**
9. **Distinguish Git desired state from Kubernetes live state.**
10. **Verify the final state after every recovery.**

The goal of troubleshooting is not simply to make the system work again.

The goal is to understand **why it failed, which component was responsible, how it recovered, and how to prevent the same class of failure in the future.**

This document complements the milestone record rather than replacing it: the project's M17 failure-testing work answers "what failure did we deliberately test, and what happened?" — this document answers "how do I diagnose this class of failure in general?"
