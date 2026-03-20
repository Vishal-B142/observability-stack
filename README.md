# observability-stack

> Production observability stack deployed to k3s via `Jenkinsfile.infra` — kube-prometheus-stack (Prometheus + Grafana + Alertmanager + Node Exporter + kube-state-metrics), Loki log aggregation, Promtail DaemonSet, and Headlamp K8s dashboard.

---

## Stack

![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-F0A622?style=flat&logo=grafana&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  app namespace — all services                                    │
│  pods emitting logs + exposing /metrics                          │
└───────────┬──────────────────────┬───────────────────────────────┘
            │ Promtail reads       │ Prometheus scrapes
            │ /var/log/containers  │ pod metrics
            ▼                      ▼
┌───────────────────────────────────────────────────────────────────┐
│  monitoring namespace                                             │
│                                                                   │
│  ┌──────────────────┐     ┌────────────────────────────────────┐  │
│  │  Promtail        │────▶│  Loki (SingleBinary mode)          │  │
│  │  DaemonSet       │     │  emptyDir at /var/loki             │  │
│  │  1 pod per node  │     │  loki-gateway  (nginx proxy :80)   │  │
│  └──────────────────┘     └────────────────────────────────────┘  │
│                                        │                          │
│  ┌─────────────────────────────────────▼──────────────────────┐   │
│  │  Grafana                                                    │   │
│  │  emptyDir (dashboards provisioned as code in values.yaml)   │   │
│  │  Datasources: Prometheus (default by chart), Loki           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│            ▲                                                       │
│  ┌─────────┴──────────┐   ┌────────────────┐   ┌───────────────┐  │
│  │  Prometheus        │   │  Alertmanager  │   │  Node         │  │
│  │  emptyDir storage  │   │                │   │  Exporter     │  │
│  │  kube-state-metrics│   │                │   │  VPS metrics  │  │
│  └────────────────────┘   └────────────────┘   └───────────────┘  │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│  headlamp namespace                                               │
│  Headlamp — web K8s dashboard (kubectl manifest, not Helm)        │
│  Token auth: kubectl create token headlamp-admin -n headlamp      │
└───────────────────────────────────────────────────────────────────┘
```

---

## Repo Structure

```
observability-stack/
├── monitoring/
│   ├── monitoring-values.yaml   # kube-prometheus-stack Helm values
│   ├── loki-values.yaml         # Loki Helm values
│   ├── promtail-values.yaml     # Promtail Helm values
│   ├── monitoring-ingress.yaml  # Traefik Ingress for Grafana
│   └── headlamp-ingress.yaml    # Traefik Ingress for Headlamp
└── README.md
```

---

## Install

### Prometheus + Grafana (kube-prometheus-stack)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f monitoring/monitoring-values.yaml \
  --atomic=false
  # NOTE: --atomic=false intentional — --wait causes 15min timeout
```

### Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install loki grafana/loki \
  -n monitoring \
  -f monitoring/loki-values.yaml \
  --atomic=false
```

### Promtail

```bash
helm upgrade --install promtail grafana/promtail \
  -n monitoring \
  -f monitoring/promtail-values.yaml \
  --atomic=false
```

### Headlamp

```bash
# Helm repo (headlamp-k8s.github.io/headlamp/) returns 404 — use direct manifest
kubectl apply -f https://raw.githubusercontent.com/headlamp-k8s/headlamp/main/kubernetes-headlamp.yaml
kubectl apply -f monitoring/headlamp-ingress.yaml --validate=false

# Get access token (8760h = 1 year)
kubectl create token headlamp-admin -n headlamp --duration=8760h
```

---

## monitoring-values.yaml (key sections)

```yaml
# Grafana — emptyDir to avoid PVC binding issues on local-path provisioner
grafana:
  persistence:
    enabled: false          # PVC caused FailedScheduling — use emptyDir
  # Dashboards provisioned as code — survive pod restarts
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: default
          folder: HM
          type: file
          options:
            path: /var/lib/grafana/dashboards/default
  # Do NOT set isDefault: true on any datasource
  # kube-prometheus-stack v82+ auto-provisions Prometheus as default internally
  # Two defaults = CrashLoopBackOff
  additionalDataSources:
    - name: Loki
      type: loki
      # Use loki-gateway service URL — NOT loki:3100
      # Loki chart v6+ deploys a gateway (nginx proxy) in front of Loki
      url: http://loki-gateway.monitoring.svc.cluster.local

# Prometheus — emptyDir storage
prometheus:
  prometheusSpec:
    storageSpec: {}           # emptyDir — no PVC needed on k3s

# Node Exporter — VPS hardware metrics (CPU, memory, disk, network)
nodeExporter:
  enabled: true

# kube-state-metrics — Kubernetes object metrics (pod counts, HPA targets)
kubeStateMetrics:
  enabled: true
```

---

## loki-values.yaml (key sections)

```yaml
loki:
  commonConfig:
    replication_factor: 1
  schemaConfig:               # Required in Loki chart v6+
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  storage:
    type: filesystem

singleBinary:
  replicas: 1
  # emptyDir at /var/loki — required even with persistence disabled
  # Loki writes index/chunks/ruler to /var/loki regardless
  extraVolumes:
    - name: loki-data
      emptyDir: {}
  extraVolumeMounts:
    - name: loki-data
      mountPath: /var/loki

persistence:
  enabled: false
```

---

## Useful Queries

### LogQL (Loki)

```logql
# All logs from a service
{namespace="app", app="gateway-service"}

# Error logs across all services
{namespace="app"} |= "ERROR"

# Logs from active slot only
{namespace="app", slot="blue"}

# Startup logs after a deploy
{namespace="app", app="auth-service"} |~ "started|listening|ready"

# Error count per service, last 1 hour
sum by (app) (count_over_time({namespace="app"} |= "ERROR" [1h]))
```

### PromQL (Prometheus)

```promql
# Current replica count per service
kube_horizontalpodautoscaler_status_current_replicas{namespace="app"}

# Services scaled above 1 replica (active scaling)
kube_horizontalpodautoscaler_status_current_replicas{namespace="app"} > 1

# CPU utilisation vs HPA threshold
kube_horizontalpodautoscaler_status_current_metrics_average_utilization{namespace="app"}

# Node memory available
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# Pod restarts (CrashLoop indicator)
rate(kube_pod_container_status_restarts_total{namespace="app"}[5m]) > 0
```

---

## Grafana Dashboards

| Dashboard | Location | What it shows |
|---|---|---|
| Kubernetes Pods | HM folder | CPU + memory per pod, filter by namespace |
| Node Exporter Full | HM folder | VPS host: CPU cores, memory, disk I/O, network |
| HPA Replicas | HM folder | Live replica counts and scaling events |
| Loki Logs | Explore | Real-time log stream with LogQL |

---

## Known Issues & Fixes

| Issue | Root Cause | Fix |
|---|---|---|
| Grafana stuck in Pending (FailedScheduling) | local-path PVC never bound — 0 capacity | Set `persistence.enabled: false` — use emptyDir |
| Grafana CrashLoopBackOff — duplicate default datasource | kube-prometheus-stack v82+ auto-provisions Prometheus default; values.yaml also set `isDefault: true` | Remove `isDefault: true` from all datasources — let the chart manage defaults |
| Loki CrashLoopBackOff — read-only filesystem | Loki writes to `/var/loki` even when `persistence.enabled: false` | Add emptyDir volume mounted at `/var/loki` via `extraVolumes` + `extraVolumeMounts` |
| Loki install fails — missing schema_config | Loki chart v6+ requires explicit `schemaConfig` block | Add `schemaConfig` with `tsdb` store, `filesystem` object store, schema `v13` |
| Grafana shows Loki datasource connection error | Loki chart v6+ deploys a gateway (nginx proxy) in front of Loki | Use `http://loki-gateway.monitoring.svc.cluster.local` — NOT `loki:3100` |
| Headlamp Helm install fails | `headlamp-k8s.github.io/headlamp/` returns 404 | Deploy via `kubectl apply` on direct manifest URL |
| Headlamp stage blocks ingress apply | Stage failure propagates downstream | Wrap in `catchError(buildResult: 'UNSTABLE')` in Jenkins |
| `helm upgrade --install` timeout (15 min) | `--wait` blocks pipeline until all pods ready | Use `--atomic=false` — submits Helm release, k8s handles rollout |

---

## Related

- [k3s-dev-staging](https://github.com/Vishal-B142/k3s-dev-staging) — cluster this stack runs on
- [jenkins-k8s-pipeline](https://github.com/Vishal-B142/jenkins-k8s-pipeline) — `Jenkinsfile.infra` that deploys this stack
- [eks-production](https://github.com/Vishal-B142/eks-production) — EKS version uses EBS + S3 backend (see runbook)
