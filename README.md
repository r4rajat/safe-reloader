# 🛡️ Safe-Reloader

> **Canary-Verified, Zero-Downtime Secret & ConfigMap Rollout Operator for Kubernetes**

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Go Version](https://img.shields.io/github/go-mod/go-version/r4rajat/safe-reloader)](https://golang.org)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-%3E%3D%201.26-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/safe-reloader)](https://artifacthub.io)

Safe-Reloader is a Kubernetes operator that prevents bad configuration changes from causing production outages. When a `Secret` or `ConfigMap` is updated, Safe-Reloader spins up an isolated shadow canary pod to verify application readiness *before* triggering rolling restarts across your workloads.

---

## 💥 The Problem: Why Blind Reloaders Fail

Popular tools like Stakater Reloader watch for `Secret` and `ConfigMap` changes and immediately trigger a rolling restart on all dependent workloads.

If an updated configuration contains an invalid token, syntax error, or bad connection string:
1. The operator blindly triggers a rolling restart.
2. New pods enter `CrashLoopBackOff` or fail their `readinessProbe`.
3. Workload capacity drops, and production goes down.
4. On-call engineers are paged *after* the incident has occurred.

```text
Standard (Blind) Reload:
[Secret / ConfigMap Updated] ───► [Blind Rolling Restart] ───► [💥 CrashLoopBackOff in Production]

Safe-Reloader Flow:
[Secret / ConfigMap Updated] ───► [Isolated Canary Pod] ───► [Probes Fail] ───► [🛡️ Rollout Blocked & Alerted]
                                                                                (Production remains 100% healthy)
```

---

## ✨ Key Features

- **Zero-Traffic Canary Isolation:** Canary pods mirror the target workload spec but strip Service selector labels. Real user traffic is never routed to unverified pods.
- **O(1) Reverse Inverted Indexing:** Uses custom `client-go` FieldIndexers to map modified Secrets/ConfigMaps to target workloads instantly without cluster-wide scanning.
- **Automated Circuit Breaking:** If the canary crashes or fails probes during the stability window, the rollout is aborted, the canary is cleaned up, and a warning event is published.
- **Dual Operating Modes:**
  - **Annotation Mode:** Drop-in replacement for Stakater Reloader (`safe-reloader.io/auto: "true"`).
  - **CRD Policy Mode:** Fine-grained governance via `SafeReloadPolicy` for timeouts, probe intervals, and notification hooks.
- **Cloud-Native Observability:** Publishes native Kubernetes Events and exports custom Prometheus metrics out of the box.

---

## 🏗️ Architecture & Reconcile Flow

```text
                 ┌────────────────────────────────┐
                 │     ConfigMap / Secret Edit    │
                 └───────────────┬────────────────┘
                                 │ Watched via client-go Informer
                                 ▼
                 ┌────────────────────────────────┐
                 │     Safe-Reloader Manager      │
                 │   - Reverse FieldIndexer (O1)  │
                 │   - Level-Triggered Engine     │
                 └───────────────┬────────────────┘
                                 │
                 ┌───────────────┴────────────────┐
                 ▼                                ▼
       [Create Shadow Canary]           [Evaluate Health Window]
     Clone Pod Spec (No Svc Labels)      Startup & Readiness Probes
                 │                                │
                 └────────────────┬───────────────┘
                                  ▼
                        ┌───────────────────┐
                        │   Probes Passed?  │
                        └─────┬───────┬─────┘
                        YES   │       │   NO
                              │       └───► [Block Rollout + Clean Canary + Emit Warning Alert]
                              ▼
                 [Trigger Safe Rolling Update]
```

---

## 🚀 Quickstart

### 1. Install Safe-Reloader

```bash
# Using kubectl
kubectl apply -f https://github.com/r4rajat/safe-reloader/releases/latest/download/install.yaml

# Or using Helm
helm repo add safe-reloader https://r4rajat.github.io/safe-reloader
helm install safe-reloader safe-reloader/safe-reloader -n safe-reloader-system --create-namespace
```

### 2. Protect a Workload (Annotation Mode)

Add the annotations to your existing `Deployment` or `StatefulSet`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
  annotations:
    safe-reloader.io/auto: "true"
    safe-reloader.io/stability-window: "30s"
    safe-reloader.io/timeout: "60s"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
        - name: app
          image: payment-service:v2.1.0
          envFrom:
            - secretRef:
                name: payment-db-credentials
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

### 3. Fine-Grained Policy (CRD Mode)

For complex workloads requiring custom validation scripts or alerting sinks:

```yaml
apiVersion: config.safereloader.io/v1alpha1
kind: SafeReloadPolicy
metadata:
  name: payment-safe-reload
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service

  watch:
    secrets:
      - payment-db-credentials
    configMaps:
      - payment-app-config

  verification:
    timeoutSeconds: 90
    stabilityWindowSeconds: 30
    failureAction: BlockAndAlert # Options: BlockAndAlert, RevertConfig

  notifications:
    onFailure:
      webhookUrl: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
```

---

## ⚙️ Configuration Reference

### Workload Annotations

| Annotation | Description | Default |
| :--- | :--- | :--- |
| `safe-reloader.io/auto` | Enables Safe-Reloader for all mounted/referenced secrets and configmaps | `"false"` |
| `safe-reloader.io/secrets` | Comma-separated list of specific secrets to track | `""` |
| `safe-reloader.io/configmaps` | Comma-separated list of specific configmaps to track | `""` |
| `safe-reloader.io/stability-window` | Duration the canary must remain healthy before triggering rollout | `"30s"` |
| `safe-reloader.io/timeout` | Maximum wait time for canary verification before timeout failure | `"60s"` |

---

## 🛠️ Local Development

### Prerequisites

- **Go:** `1.24+`
- **Docker** / **k3d**
- **Kubebuilder:** `v4.0+`
- **Make**

### Setup & Run Locally

```bash
# 1. Clone the repository
git clone https://github.com/r4rajat/safe-reloader.git
cd safe-reloader

# 2. Create a local k3d cluster
k3d cluster create safe-reloader-dev --agents 1

# 3. Install CRDs into the cluster
make install

# 4. Run the controller locally against your cluster
make run
```

### Running Tests

```bash
# Run unit & envtest integration tests
make test
```

---

## 🗺️ Roadmap

- [ ] **v0.1.0 (MVP):**
  - Deployment support with annotation-driven mode
  - Ephemeral shadow canary pod lifecycle management
  - Reverse FieldIndexer cache for $O(1)$ secret/configmap lookups
  - Native Kubernetes event emission
- [ ] **v0.2.0:**
  - StatefulSet and DaemonSet support
  - `SafeReloadPolicy` CRD implementation
  - Custom HTTP synthetic validation probes
- [ ] **v0.3.0:**
  - Mutating and Validating Admission Webhooks
  - Multi-tenant RBAC profiles
- [ ] **v0.4.0:**
  - Webhook alerting sinks (Slack, Microsoft Teams, PagerDuty)
  - Automatic secret reversion option
- [ ] **v1.0.0:**
  - Production-ready Helm chart & Operator Lifecycle Manager (OLM) bundle
  - Official Artifact Hub publication

---

## 🤝 Contributing

We welcome contributions from the community. Please review the [CONTRIBUTING.md](CONTRIBUTING.md) guide before submitting pull requests.

---

## 📄 License

Licensed under the **Apache License, Version 2.0**. See the [LICENSE](LICENSE) file for details.
