# devops-sentiment-k8s-gitops

> **GitOps deployment platform** for the `ml-sentiment-app` — Kubernetes manifests, ArgoCD, Kustomize overlays, canary rollouts via Argo Rollouts, and a full Prometheus + Grafana observability stack.

Part of a two-repo DevOps platform built for **CISC 814 — DevOps and Continuous Delivery** at Queen's University (Summer 2026).
The application source code and CI pipeline live in the companion repo: [`ml-sentiment-cicd`](https://github.com/Mazen-Bahgat/ml-sentiment-cicd).

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Deployment Guide](#deployment-guide)
  - [1. Kubernetes Manifests](#1-kubernetes-manifests)
  - [2. Kustomize Overlays](#2-kustomize-overlays)
  - [3. ArgoCD Application](#3-argocd-application)
  - [4. Canary Deployment with Argo Rollouts](#4-canary-deployment-with-argo-rollouts)
  - [5. Prometheus Metrics & ServiceMonitor](#5-prometheus-metrics--servicemonitor)
  - [6. Grafana Dashboard](#6-grafana-dashboard)
- [End-to-End Deployment Flow](#end-to-end-deployment-flow)
- [Runbook](#runbook)
- [Triggering a New Deployment](#triggering-a-new-deployment)
- [Service Endpoints](#service-endpoints)
- [Related Repository](#related-repository)
- [Team](#team)

---

## Overview

This repo is the **single source of truth** for how `ml-sentiment-app` runs in Kubernetes. No-one applies manifests by hand — ArgoCD continuously reconciles whatever is in this repo against the live cluster state.

Key capabilities:

- **GitOps sync loop** — ArgoCD polls the repo every ~30 seconds; any drift is auto-healed
- **Progressive delivery** — Argo Rollouts shifts traffic in five 20 % increments with 30-second pauses between each step
- **Environment promotion** — Kustomize base + production overlay cleanly separates shared config from environment-specific patches
- **Full observability** — Prometheus scrapes `/metrics` every 15 seconds; four Grafana panels give real-time visibility into request rate, errors, latency, and pod resource usage

---

## Architecture

![CI/CD Pipeline Architecture](https://raw.githubusercontent.com/Mazen-Bahgat/devops-sentiment-k8s-gitops/main/devops-diagram.svg)

> **GitOps deployment platform** for the `ml-sentiment-app` — Kubernetes manifests, ArgoCD, Kustomize overlays, canary rollouts via Argo Rollouts, and a full Prometheus + Grafana observability stack.

Part of a two-repo DevOps platform built for **CISC 814 — DevOps and Continuous Delivery** at Queen's University (Summer 2026).
The application source code and CI pipeline live in the companion repo: [`ml-sentiment-cicd`](https://github.com/Mazen-Bahgat/ml-sentiment-cicd).

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Deployment Guide](#deployment-guide)
  - [1. Kubernetes Manifests](#1-kubernetes-manifests)
  - [2. Kustomize Overlays](#2-kustomize-overlays)
  - [3. ArgoCD Application](#3-argocd-application)
  - [4. Canary Deployment with Argo Rollouts](#4-canary-deployment-with-argo-rollouts)
  - [5. Prometheus Metrics & ServiceMonitor](#5-prometheus-metrics--servicemonitor)
  - [6. Grafana Dashboard](#6-grafana-dashboard)
- [End-to-End Deployment Flow](#end-to-end-deployment-flow)
- [Runbook](#runbook)
- [Triggering a New Deployment](#triggering-a-new-deployment)
- [Service Endpoints](#service-endpoints)
- [Related Repository](#related-repository)
- [Team](#team)

---

## Overview

This repo is the **single source of truth** for how `ml-sentiment-app` runs in Kubernetes. No-one applies manifests by hand — ArgoCD continuously reconciles whatever is in this repo against the live cluster state.

Key capabilities:

- **GitOps sync loop** — ArgoCD polls the repo every ~30 seconds; any drift is auto-healed
- **Progressive delivery** — Argo Rollouts shifts traffic in five 20 % increments with 30-second pauses between each step
- **Environment promotion** — Kustomize base + production overlay cleanly separates shared config from environment-specific patches
- **Full observability** — Prometheus scrapes `/metrics` every 15 seconds; four Grafana panels give real-time visibility into request rate, errors, latency, and pod resource usage

---

## Architecture

```
                    ┌─────────────────────────────────────┐
                    │     ml-sentiment-cicd (CI repo)      │
                    │  Gitea Actions → Docker build/push   │
                    │  Image: registry.local:5000/         │
                    │         ml-sentiment-app:<git-sha>   │
                    └────────────────┬────────────────────┘
                                     │ image tag committed
                                     ▼
┌────────────────────────────────────────────────────────────────────┐
│                   This repo (GitOps source of truth)               │
│                                                                    │
│  argocd-application.yaml                                           │
│  apps/ml-sentiment-app/                                            │
│  ├── base/           (configmap, rollout, services, kustomization) │
│  └── overlays/production/  (replicas=4, CPU/mem limits, pinned SHA)│
│  monitoring/         (ServiceMonitor, Grafana dashboard ConfigMap)  │
└────────────────────────────────┬───────────────────────────────────┘
                                 │ polls every ~30s (GitOps sync loop)
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│                          k3d Cluster                               │
│                                                                    │
│  ┌──────────┐   renders    ┌───────────┐   apply    ┌──────────┐  │
│  │  ArgoCD  │ ──────────→  │ Kustomize │ ─────────→ │  Argo    │  │
│  │ selfHeal │              │ base +    │            │ Rollout  │  │
│  │ + prune  │              │ production│            │ canary   │  │
│  └──────────┘              └───────────┘            └────┬─────┘  │
│                                                          │        │
│              ┌───────────────────────┬──────────────────┤        │
│              ▼                       ▼                   ▼        │
│       Stable Pods             Canary Pods           ConfigMap     │
│    ml-sentiment-app (old)  ml-sentiment-app (new)  APP_ENV        │
│       80% traffic               20% traffic        LOG_LEVEL      │
│              │                       │                            │
│              └──────────┬────────────┘                            │
│                         │ expose /metrics                         │
│                         ▼                                         │
│              ┌──────────────────┐    scrape every 15s             │
│              │ ServiceMonitor   │ ──────────────────────────────→ │
│              │ CRD              │                                  │
│              └──────────────────┘          ┌─────────────────┐   │
│                                            │   Prometheus    │   │
│                                            │ prediction_*    │   │
│                                            └────────┬────────┘   │
│                                                     │ PromQL     │
│                                                     ▼            │
│                                            ┌─────────────────┐   │
│                                            │     Grafana     │   │
│                                            │ 4 panels:       │   │
│                                            │ rate/error/     │   │
│                                            │ latency/CPU+mem │   │
│                                            └─────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

**Canary traffic steps:** `20% → 40% → 60% → 80% → 100%` with 30 s pauses

---

## Repository Structure

```
devops-sentiment-k8s-gitops/
├── argocd-application.yaml                     # ArgoCD Application CRD — points at production overlay
│
├── apps/
│   └── ml-sentiment-app/
│       ├── base/
│       │   ├── configmap.yml                   # APP_ENV=production, LOG_LEVEL=info
│       │   ├── deployment.yml                  # Base Deployment (testing/dev only)
│       │   ├── rollout.yaml                    # Argo Rollout + 9-step canary strategy
│       │   ├── service.yaml                    # Stable ClusterIP service (port 80 → 8000)
│       │   ├── canary-service.yaml             # Canary ClusterIP service
│       │   └── kustomization.yaml              # Base resource list + common labels
│       │
│       └── overlays/
│           └── production/
│               └── kustomization.yaml          # Patches: replicas=4, CPU/mem limits, pinned image SHA
│
└── monitoring/
    ├── servicemonitor.yaml                     # Prometheus ServiceMonitor — scrape /metrics every 15s
    ├── grafana-dashboard-configmap.yaml        # Grafana dashboard provisioned via sidecar
    └── kustomization.yaml
```

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| `kubectl` | ≥ 1.27 | Cluster interaction |
| `kustomize` | ≥ 5.0 | Manifest rendering |
| `kubectl argo rollouts` plugin | ≥ 1.6 | Rollout inspection and promotion |
| ArgoCD | ≥ 2.4 | GitOps controller |
| kube-prometheus-stack | any | Prometheus operator + Grafana |
| k3d | ≥ 5.0 | Local cluster (used in this project) |

**Set your kubeconfig:**

```bash
export KUBECONFIG=/home/localuser/assignment-1/cisc814-s26-assignment1/\
cisc814-s26-assignments-main/infrastructure/k3d/kubeconfigs/cisc814-team-4.yaml
```

---

## Deployment Guide

### 1. Kubernetes Manifests

The base layer in `apps/ml-sentiment-app/base/` defines all resources the application needs.

| File | Kind | Key settings |
|---|---|---|
| `configmap.yml` | ConfigMap | `APP_ENV=production`, `LOG_LEVEL=info` |
| `deployment.yml` | Deployment | 2 replicas, 100m/128Mi requests, 500m/512Mi limits, `/health` probes — **testing only** |
| `rollout.yaml` | Rollout | 5 replicas (base), canary strategy, references `stableService` and `canaryService` |
| `service.yaml` | Service | ClusterIP, port `80 → 8000`, named port `http` |
| `canary-service.yaml` | Service | Identical to `service.yaml`, named `ml-sentiment-app-canary` |

Apply the base directly (dev/test):

```bash
kubectl apply -k apps/ml-sentiment-app/base/
```

### 2. Kustomize Overlays

The production overlay patches three fields over the base:

| Field | Base value | Production value |
|---|---|---|
| `spec.replicas` (Rollout) | 5 | **4** |
| CPU limit | 500m | **1000m** |
| Memory limit | 512Mi | **1Gi** |
| Image tag | `latest` | **Pinned Git SHA** (e.g. `213fc639…`) |

Preview the rendered manifests:

```bash
kubectl apply -k apps/ml-sentiment-app/overlays/production/ --dry-run=client
```

Apply manually (or let ArgoCD do it):

```bash
kubectl apply -k apps/ml-sentiment-app/overlays/production/
```

### 3. ArgoCD Application

`argocd-application.yaml` registers the application with ArgoCD. It points at the production overlay on the `main` branch.

| Property | Value |
|---|---|
| Source repo | `http://cisc814-gitea:3000/cisc814-gitea/team-4-gitops.git` |
| Target path | `apps/ml-sentiment-app/overlays/production` |
| Branch | `main` |
| Sync policy | `selfHeal: true`, `prune: true`, `CreateNamespace=true` |
| Poll interval | ~30 seconds |

Apply the ArgoCD Application:

```bash
kubectl apply -f argocd-application.yaml
```

Check sync status:

```bash
kubectl get application ml-sentiment-app -n argocd
```

ArgoCD UI: `https://localhost:8400`

### 4. Canary Deployment with Argo Rollouts

The `rollout.yaml` replaces the standard `Deployment` with a nine-step canary strategy:

| Steps | Action | Canary traffic |
|---|---|---|
| 1 + 2 | `setWeight 20` then `pause 30s` | 20 % |
| 3 + 4 | `setWeight 40` then `pause 30s` | 40 % |
| 5 + 6 | `setWeight 60` then `pause 30s` | 60 % |
| 7 + 8 | `setWeight 80` then `pause 30s` | 80 % |
| 9 | `setWeight 100` | 100 % stable |

Watch a rollout in progress:

```bash
kubectl argo rollouts get rollout ml-sentiment-app --watch
```

Manually promote a paused rollout:

```bash
kubectl argo rollouts promote ml-sentiment-app
```

Abort and roll back:

```bash
kubectl argo rollouts abort ml-sentiment-app
kubectl argo rollouts undo ml-sentiment-app
```

### 5. Prometheus Metrics & ServiceMonitor

Three custom metrics are exposed by the application at `/metrics`:

| Metric | Type | Labels |
|---|---|---|
| `prediction_requests_total` | Counter | `method`, `endpoint`, `status` |
| `predictions_per_label_total` | Counter | `label` |
| `prediction_request_duration_seconds` | Histogram | `method`, `endpoint` |

`monitoring/servicemonitor.yaml` instructs the Prometheus operator to scrape `/metrics` every **15 seconds**.
It matches the label `app.kubernetes.io/name: ml-sentiment-app` on port `http`, and carries `release: kube-prometheus-stack` so the operator discovers it automatically.

Apply monitoring resources:

```bash
kubectl apply -k monitoring/
```

Verify the scrape target is UP:

```bash
# Port-forward Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090
# Then open http://localhost:9090/targets
```

### 6. Grafana Dashboard

The dashboard is defined as a ConfigMap in `monitoring/grafana-dashboard-configmap.yaml` with the label `grafana_dashboard: "1"`. The Grafana sidecar auto-provisions it on pod startup — no manual import needed.

| Panel | PromQL | What it shows |
|---|---|---|
| Request Rate | `rate(prediction_requests_total[5m])` | Requests/sec (5-min window) |
| Error Rate (5xx) | `rate(prediction_requests_total{status=~"5.."}[5m])` | HTTP 5xx errors/sec |
| Latency p50/p95/p99 | `histogram_quantile(0.50/0.95/0.99, rate(prediction_request_duration_seconds_bucket[5m]))` | Prediction latency percentiles |
| CPU/Memory per pod | `rate(container_cpu_usage_seconds_total{pod=~"ml-sentiment-app.*"}[5m])` + `container_memory_usage_bytes{pod=~"ml-sentiment-app.*"}` | Per-pod resource usage |

Grafana UI: `http://localhost:8500`

---

## End-to-End Deployment Flow

```
1. Developer pushes a code change to ml-sentiment-cicd
       ↓
2. Gitea Actions runs: Ruff → Mypy → pytest → Docker build → Docker push → Syft SBOM
       ↓
3. Image tagged :<git-sha> pushed to registry.local:5000
       ↓
4. Image tag updated in overlays/production/kustomization.yaml
   and committed to this (GitOps) repo
       ↓
5. ArgoCD detects the diff within ~30s
   → runs kustomize build on the production overlay
       ↓
6. ArgoCD applies rendered manifests
   → triggers Argo Rollout
       ↓
7. Argo Rollouts steps traffic:
   20% → 40% → 60% → 80% → 100%  (30s pauses)
       ↓
8. Prometheus scrapes /metrics every 15s
   → Grafana dashboard updates in real time
```

---

## Runbook

```bash
# ── Access & info ─────────────────────────────────────────────────────
# SSH tunnel to expose all UIs locally
ssh -L 3000:localhost:3000 \
    -L 8400:localhost:8400 \
    -L 8500:localhost:8500 \
    -L 8600:localhost:8600 \
    localuser@130.15.5.209

# Retrieve all port info
bash manage-cluster.sh info team-4

# Gitea        → http://localhost:3000
# ArgoCD       → https://localhost:8400
# Grafana      → http://localhost:8500
# Prometheus   → http://localhost:8600

# ── Cluster state ────────────────────────────────────────────────────
kubectl get all -n default
kubectl argo rollouts get rollout ml-sentiment-app
kubectl get servicemonitors -n monitoring
kubectl get configmaps -n monitoring | grep grafana
kubectl get application ml-sentiment-app -n argocd

# ── ArgoCD ───────────────────────────────────────────────────────────
kubectl get application ml-sentiment-app -n argocd -o yaml
# Force a manual sync
argocd app sync ml-sentiment-app
```

---

## Triggering a New Deployment

```bash
# 1. Push a code change to ml-sentiment-cicd
#    CI builds and pushes the new image with a new SHA tag

# 2. Update the image tag in the production overlay
cd apps/ml-sentiment-app/overlays/production/
# Edit kustomization.yaml → newTag: <new-sha>
git add .
git commit -m "chore: bump image to <new-sha>"
git push

# 3. ArgoCD auto-syncs within ~30s and triggers Argo Rollout

# 4. Watch the rollout
kubectl argo rollouts get rollout ml-sentiment-app --watch
```

---

## Service Endpoints

| Service | URL |
|---|---|
| Gitea (app repo) | `http://localhost:3000/cisc814-gitea/ml-sentiment-app` |
| Gitea (gitops repo) | `http://localhost:3000/cisc814-gitea/team-4-gitops` |
| Container Registry | `localhost:5001` (host) / `registry.local:5000` (in-cluster) |
| ArgoCD UI | `https://localhost:8400` |
| Grafana UI | `http://localhost:8500` |
| Prometheus UI | `http://localhost:8600` |
| VM | `localuser@130.15.5.209` |

---

## Related Repository

| Repo | Purpose |
|---|---|
| [`ml-sentiment-cicd`](https://github.com/Mazen-Bahgat/ml-sentiment-cicd) | Application source code, FastAPI service, CI pipeline, Dockerfile, SBOM |
| **This repo** — [`devops-sentiment-k8s-gitops`](https://github.com/Mazen-Bahgat/devops-sentiment-k8s-gitops) | Kubernetes manifests, ArgoCD, Kustomize, Argo Rollouts, Prometheus, Grafana |

---

## Team

**CISC 814 — Team 4 · Queen's University, Summer 2026**

| Member | Student ID | Part A Focus | Part B Focus |
|---|---|---|---|
| Hana Amrya | 25tvtm | Repo setup, branch protection, Ruff linting | Cluster setup, base K8s manifests, Grafana latency panel |
| Hassan Albattra | 25vhsj | Mypy type checking, pytest + coverage | Rollout resource, production overlay, ServiceMonitor |
| Mazen Bahgat | 25dtg4 | Infrastructure, Dockerfile | GitOps repo, ArgoCD, canary step design |
| Mariam Mousa | 25qns4 | Docker build + push, Syft SBOM | Prometheus metrics, canary traffic testing, Grafana error panel |

---

> **Demo video:** [Full end-to-end deployment walkthrough](https://drive.google.com/file/d/1JrWXD9OctY18s7I3XvVHQgY5bruHw9RI/view?usp=sharing) — CI trigger → Docker push → ArgoCD sync → canary rollout → live Grafana dashboard (10 min).]

**Canary traffic steps:** `20% → 40% → 60% → 80% → 100%` with 30 s pauses

---

## Repository Structure

```
devops-sentiment-k8s-gitops/
├── argocd-application.yaml                     # ArgoCD Application CRD — points at production overlay
│
├── apps/
│   └── ml-sentiment-app/
│       ├── base/
│       │   ├── configmap.yml                   # APP_ENV=production, LOG_LEVEL=info
│       │   ├── deployment.yml                  # Base Deployment (testing/dev only)
│       │   ├── rollout.yaml                    # Argo Rollout + 9-step canary strategy
│       │   ├── service.yaml                    # Stable ClusterIP service (port 80 → 8000)
│       │   ├── canary-service.yaml             # Canary ClusterIP service
│       │   └── kustomization.yaml              # Base resource list + common labels
│       │
│       └── overlays/
│           └── production/
│               └── kustomization.yaml          # Patches: replicas=4, CPU/mem limits, pinned image SHA
│
└── monitoring/
    ├── servicemonitor.yaml                     # Prometheus ServiceMonitor — scrape /metrics every 15s
    ├── grafana-dashboard-configmap.yaml        # Grafana dashboard provisioned via sidecar
    └── kustomization.yaml
```

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| `kubectl` | ≥ 1.27 | Cluster interaction |
| `kustomize` | ≥ 5.0 | Manifest rendering |
| `kubectl argo rollouts` plugin | ≥ 1.6 | Rollout inspection and promotion |
| ArgoCD | ≥ 2.4 | GitOps controller |
| kube-prometheus-stack | any | Prometheus operator + Grafana |
| k3d | ≥ 5.0 | Local cluster (used in this project) |

**Set your kubeconfig:**

```bash
export KUBECONFIG=/home/localuser/assignment-1/cisc814-s26-assignment1/\
cisc814-s26-assignments-main/infrastructure/k3d/kubeconfigs/cisc814-team-4.yaml
```

---

## Deployment Guide

### 1. Kubernetes Manifests

The base layer in `apps/ml-sentiment-app/base/` defines all resources the application needs.

| File | Kind | Key settings |
|---|---|---|
| `configmap.yml` | ConfigMap | `APP_ENV=production`, `LOG_LEVEL=info` |
| `deployment.yml` | Deployment | 2 replicas, 100m/128Mi requests, 500m/512Mi limits, `/health` probes — **testing only** |
| `rollout.yaml` | Rollout | 5 replicas (base), canary strategy, references `stableService` and `canaryService` |
| `service.yaml` | Service | ClusterIP, port `80 → 8000`, named port `http` |
| `canary-service.yaml` | Service | Identical to `service.yaml`, named `ml-sentiment-app-canary` |

Apply the base directly (dev/test):

```bash
kubectl apply -k apps/ml-sentiment-app/base/
```

### 2. Kustomize Overlays

The production overlay patches three fields over the base:

| Field | Base value | Production value |
|---|---|---|
| `spec.replicas` (Rollout) | 5 | **4** |
| CPU limit | 500m | **1000m** |
| Memory limit | 512Mi | **1Gi** |
| Image tag | `latest` | **Pinned Git SHA** (e.g. `213fc639…`) |

Preview the rendered manifests:

```bash
kubectl apply -k apps/ml-sentiment-app/overlays/production/ --dry-run=client
```

Apply manually (or let ArgoCD do it):

```bash
kubectl apply -k apps/ml-sentiment-app/overlays/production/
```

### 3. ArgoCD Application

`argocd-application.yaml` registers the application with ArgoCD. It points at the production overlay on the `main` branch.

| Property | Value |
|---|---|
| Source repo | `http://cisc814-gitea:3000/cisc814-gitea/team-4-gitops.git` |
| Target path | `apps/ml-sentiment-app/overlays/production` |
| Branch | `main` |
| Sync policy | `selfHeal: true`, `prune: true`, `CreateNamespace=true` |
| Poll interval | ~30 seconds |

Apply the ArgoCD Application:

```bash
kubectl apply -f argocd-application.yaml
```

Check sync status:

```bash
kubectl get application ml-sentiment-app -n argocd
```

ArgoCD UI: `https://localhost:8400`

### 4. Canary Deployment with Argo Rollouts

The `rollout.yaml` replaces the standard `Deployment` with a nine-step canary strategy:

| Steps | Action | Canary traffic |
|---|---|---|
| 1 + 2 | `setWeight 20` then `pause 30s` | 20 % |
| 3 + 4 | `setWeight 40` then `pause 30s` | 40 % |
| 5 + 6 | `setWeight 60` then `pause 30s` | 60 % |
| 7 + 8 | `setWeight 80` then `pause 30s` | 80 % |
| 9 | `setWeight 100` | 100 % stable |

Watch a rollout in progress:

```bash
kubectl argo rollouts get rollout ml-sentiment-app --watch
```

Manually promote a paused rollout:

```bash
kubectl argo rollouts promote ml-sentiment-app
```

Abort and roll back:

```bash
kubectl argo rollouts abort ml-sentiment-app
kubectl argo rollouts undo ml-sentiment-app
```

### 5. Prometheus Metrics & ServiceMonitor

Three custom metrics are exposed by the application at `/metrics`:

| Metric | Type | Labels |
|---|---|---|
| `prediction_requests_total` | Counter | `method`, `endpoint`, `status` |
| `predictions_per_label_total` | Counter | `label` |
| `prediction_request_duration_seconds` | Histogram | `method`, `endpoint` |

`monitoring/servicemonitor.yaml` instructs the Prometheus operator to scrape `/metrics` every **15 seconds**.
It matches the label `app.kubernetes.io/name: ml-sentiment-app` on port `http`, and carries `release: kube-prometheus-stack` so the operator discovers it automatically.

Apply monitoring resources:

```bash
kubectl apply -k monitoring/
```

Verify the scrape target is UP:

```bash
# Port-forward Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090
# Then open http://localhost:9090/targets
```

### 6. Grafana Dashboard

The dashboard is defined as a ConfigMap in `monitoring/grafana-dashboard-configmap.yaml` with the label `grafana_dashboard: "1"`. The Grafana sidecar auto-provisions it on pod startup — no manual import needed.

| Panel | PromQL | What it shows |
|---|---|---|
| Request Rate | `rate(prediction_requests_total[5m])` | Requests/sec (5-min window) |
| Error Rate (5xx) | `rate(prediction_requests_total{status=~"5.."}[5m])` | HTTP 5xx errors/sec |
| Latency p50/p95/p99 | `histogram_quantile(0.50/0.95/0.99, rate(prediction_request_duration_seconds_bucket[5m]))` | Prediction latency percentiles |
| CPU/Memory per pod | `rate(container_cpu_usage_seconds_total{pod=~"ml-sentiment-app.*"}[5m])` + `container_memory_usage_bytes{pod=~"ml-sentiment-app.*"}` | Per-pod resource usage |

Grafana UI: `http://localhost:8500`

---

## End-to-End Deployment Flow

```
1. Developer pushes a code change to ml-sentiment-cicd
       ↓
2. Gitea Actions runs: Ruff → Mypy → pytest → Docker build → Docker push → Syft SBOM
       ↓
3. Image tagged :<git-sha> pushed to registry.local:5000
       ↓
4. Image tag updated in overlays/production/kustomization.yaml
   and committed to this (GitOps) repo
       ↓
5. ArgoCD detects the diff within ~30s
   → runs kustomize build on the production overlay
       ↓
6. ArgoCD applies rendered manifests
   → triggers Argo Rollout
       ↓
7. Argo Rollouts steps traffic:
   20% → 40% → 60% → 80% → 100%  (30s pauses)
       ↓
8. Prometheus scrapes /metrics every 15s
   → Grafana dashboard updates in real time
```

---

## Runbook

```bash
# ── Access & info ─────────────────────────────────────────────────────
# SSH tunnel to expose all UIs locally
ssh -L 3000:localhost:3000 \
    -L 8400:localhost:8400 \
    -L 8500:localhost:8500 \
    -L 8600:localhost:8600 \
    localuser@130.15.5.209

# Retrieve all port info
bash manage-cluster.sh info team-4

# Gitea        → http://localhost:3000
# ArgoCD       → https://localhost:8400
# Grafana      → http://localhost:8500
# Prometheus   → http://localhost:8600

# ── Cluster state ────────────────────────────────────────────────────
kubectl get all -n default
kubectl argo rollouts get rollout ml-sentiment-app
kubectl get servicemonitors -n monitoring
kubectl get configmaps -n monitoring | grep grafana
kubectl get application ml-sentiment-app -n argocd

# ── ArgoCD ───────────────────────────────────────────────────────────
kubectl get application ml-sentiment-app -n argocd -o yaml
# Force a manual sync
argocd app sync ml-sentiment-app
```

---

## Triggering a New Deployment

```bash
# 1. Push a code change to ml-sentiment-cicd
#    CI builds and pushes the new image with a new SHA tag

# 2. Update the image tag in the production overlay
cd apps/ml-sentiment-app/overlays/production/
# Edit kustomization.yaml → newTag: <new-sha>
git add .
git commit -m "chore: bump image to <new-sha>"
git push

# 3. ArgoCD auto-syncs within ~30s and triggers Argo Rollout

# 4. Watch the rollout
kubectl argo rollouts get rollout ml-sentiment-app --watch
```

---

## Service Endpoints

| Service | URL |
|---|---|
| Gitea (app repo) | `http://localhost:3000/cisc814-gitea/ml-sentiment-app` |
| Gitea (gitops repo) | `http://localhost:3000/cisc814-gitea/team-4-gitops` |
| Container Registry | `localhost:5001` (host) / `registry.local:5000` (in-cluster) |
| ArgoCD UI | `https://localhost:8400` |
| Grafana UI | `http://localhost:8500` |
| Prometheus UI | `http://localhost:8600` |
| VM | `localuser@130.15.5.209` |

---

## Related Repository

| Repo | Purpose |
|---|---|
| [`ml-sentiment-cicd`](https://github.com/Mazen-Bahgat/ml-sentiment-cicd) | Application source code, FastAPI service, CI pipeline, Dockerfile, SBOM |
| **This repo** — [`devops-sentiment-k8s-gitops`](https://github.com/Mazen-Bahgat/devops-sentiment-k8s-gitops) | Kubernetes manifests, ArgoCD, Kustomize, Argo Rollouts, Prometheus, Grafana |

---

## Team

**CISC 814 — Team 4 · Queen's University, Summer 2026**

| Member | Student ID | Part A Focus | Part B Focus |
|---|---|---|---|
| Hana Amrya | 25tvtm | Repo setup, branch protection, Ruff linting | Cluster setup, base K8s manifests, Grafana latency panel |
| Hassan Albattra | 25vhsj | Mypy type checking, pytest + coverage | Rollout resource, production overlay, ServiceMonitor |
| Mazen Bahgat | 25dtg4 | Infrastructure, Dockerfile | GitOps repo, ArgoCD, canary step design |
| Mariam Mousa | 25qns4 | Docker build + push, Syft SBOM | Prometheus metrics, canary traffic testing, Grafana error panel |

---

> **Demo video:** [Full end-to-end deployment walkthrough](https://drive.google.com/file/d/1JrWXD9OctY18s7I3XvVHQgY5bruHw9RI/view?usp=sharing) — CI trigger → Docker push → ArgoCD sync → canary rollout → live Grafana dashboard (10 min).
