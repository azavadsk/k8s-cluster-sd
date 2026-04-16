# Loki + Promtail Log Aggregation — Installation Guide

## Overview

This guide covers setting up centralized log aggregation for the Kubernetes cluster using **Loki** and **Promtail**, integrated with the existing Grafana instance. Logs are stored in **RustFS** (S3-compatible object storage).

**Stack:**
- Loki v3.6.7 — log aggregation and storage backend
- Promtail v3.5.1 — log collector DaemonSet (runs on every node)
- Grafana — already running, Loki added as datasource
- RustFS — S3-compatible storage for log chunks (`loki` bucket)

**How it works:**

```
Pod stdout/stderr
       │
       ▼
Promtail DaemonSet          ← runs on every node, tails /var/log/pods/
       │  pushes via HTTP
       ▼
Loki Gateway (nginx)        ← single entry point for reads and writes
       │
       ▼
Loki (SingleBinary)         ← indexes and stores logs
       │  chunks stored in
       ▼
RustFS s3://loki            ← 192.168.1.246:9000
       │
       ▼
Grafana → Explore → Loki    ← query logs via LogQL
```

---

## Prerequisites

- Kubernetes cluster running with Helm v3+
- Grafana already deployed (via kube-prometheus-stack in `monitoring` namespace)
- RustFS running at `192.168.1.246:9000` with credentials `rustfsadmin / rustfsadmin`
- Grafana Helm repo added: `helm repo add grafana https://grafana.github.io/helm-charts`

---

## Step 1 — Create the Loki Bucket in RustFS

Run from your local machine:

```bash
AWS_ACCESS_KEY_ID=rustfsadmin \
AWS_SECRET_ACCESS_KEY=rustfsadmin \
AWS_DEFAULT_REGION=us-east-1 \
  aws s3 mb s3://loki --endpoint-url http://192.168.1.246:9000
```

Verify:

```bash
AWS_ACCESS_KEY_ID=rustfsadmin \
AWS_SECRET_ACCESS_KEY=rustfsadmin \
AWS_DEFAULT_REGION=us-east-1 \
  aws s3 ls --endpoint-url http://192.168.1.246:9000
```

---

## Step 2 — Install Loki

Create `loki-values.yaml`:

```yaml
loki:
  commonConfig:
    replication_factor: 1
  auth_enabled: false

  storage:
    type: s3
    bucketNames:
      chunks: loki
      ruler: loki
      admin: loki
    s3:
      endpoint: http://192.168.1.246:9000
      region: us-east-1
      bucketnames: loki
      access_key_id: rustfsadmin
      secret_access_key: rustfsadmin
      s3forcepathstyle: true
      insecure: true

  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

deploymentMode: SingleBinary

singleBinary:
  replicas: 1
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: "1"
      memory: 512Mi

# Disable components not needed in SingleBinary mode
backend:
  replicas: 0
read:
  replicas: 0
write:
  replicas: 0
ingester:
  replicas: 0
querier:
  replicas: 0
queryFrontend:
  replicas: 0
queryScheduler:
  replicas: 0
distributor:
  replicas: 0
compactor:
  replicas: 0
patternIngester:
  replicas: 0
indexGateway:
  replicas: 0
bloomCompactor:
  replicas: 0
bloomGateway:
  replicas: 0

# Disable minio (using RustFS instead)
minio:
  enabled: false

# Disable self-monitoring to keep it lightweight
monitoring:
  selfMonitoring:
    enabled: false
    grafanaAgent:
      installOperator: false
  lokiCanary:
    enabled: false

test:
  enabled: false
```

Install:

```bash
helm install loki grafana/loki \
  --namespace monitoring \
  --values loki-values.yaml
```

Wait for Loki to be ready:

```bash
kubectl -n monitoring rollout status statefulset/loki --timeout=120s
```

---

## Step 3 — Install Promtail

Promtail runs as a DaemonSet — one pod per node — and tails all container logs from `/var/log/pods/`.

```bash
helm install promtail grafana/promtail \
  --namespace monitoring \
  --set "config.clients[0].url=http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push"
```

Wait for Promtail to be ready:

```bash
kubectl -n monitoring rollout status daemonset/promtail --timeout=120s
```

---

## Step 4 — Add Loki Datasource to Grafana

Port-forward Grafana and add the datasource via API:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 &

curl -s -X POST "http://admin:1234@127.0.0.1:3000/api/datasources" \
  -H "Content-Type: application/json" \
  --data-raw '{
    "name": "Loki",
    "type": "loki",
    "url": "http://loki-gateway.monitoring.svc.cluster.local",
    "access": "proxy",
    "isDefault": false
  }'

# Stop port-forward when done
kill %1
```

Or add it manually in Grafana UI:
1. Open `https://grafana.local`
2. Go to **Connections → Data Sources → Add new**
3. Select **Loki**
4. Set URL to `http://loki-gateway.monitoring.svc.cluster.local`
5. Click **Save & Test**

---

## Step 5 — Verify

Check all pods are running:

```bash
kubectl -n monitoring get pods | grep -E "loki|promtail"
```

Expected output:

```
loki-0                   2/2     Running   ...
loki-chunks-cache-0      2/2     Running   ...
loki-gateway-xxx         1/1     Running   ...
loki-results-cache-0     2/2     Running   ...
promtail-xxxxx           1/1     Running   ...   (one per node)
```

Verify logs are flowing into RustFS:

```bash
AWS_ACCESS_KEY_ID=rustfsadmin \
AWS_SECRET_ACCESS_KEY=rustfsadmin \
AWS_DEFAULT_REGION=us-east-1 \
  aws s3 ls s3://loki --recursive --endpoint-url http://192.168.1.246:9000 | head -20
```

---

## Querying Logs in Grafana

1. Open `https://grafana.local`
2. Go to **Explore** (compass icon)
3. Select **Loki** as the datasource
4. Use LogQL to query logs

### Common LogQL Queries

```logql
# All logs from a namespace
{namespace="jenkins"}

# All logs from a specific pod
{pod="jenkins-0"}

# All logs from a specific container
{container="argocd-server"}

# Filter by log level
{namespace="monitoring"} |= "error"
{namespace="monitoring"} |= "warn"

# Pods in crash state
{namespace=~".+"} |= "OOMKilled"
{namespace=~".+"} |= "CrashLoopBackOff"

# Logs from all pods across the cluster
{namespace=~".+"}

# Count error rate over time
count_over_time({namespace="jenkins"} |= "error" [5m])
```

---

## Installed Components

| Component | Namespace | Version | Purpose |
|-----------|-----------|---------|---------|
| loki | monitoring | 3.6.7 | Log storage and querying |
| loki-gateway | monitoring | — | nginx proxy for Loki API |
| loki-chunks-cache | monitoring | — | In-memory cache for chunks |
| loki-results-cache | monitoring | — | In-memory cache for queries |
| promtail | monitoring | 3.5.1 | Log collector (DaemonSet) |

---

## Storage

Logs are stored in RustFS object storage:

| Property | Value |
|----------|-------|
| Endpoint | http://192.168.1.246:9000 |
| Bucket | loki |
| Credentials | rustfsadmin / rustfsadmin |
| Path style | Force path style enabled |

Loki writes three types of data to the bucket:
- **chunks** — compressed log data
- **index** — TSDB index files
- **ruler** — alerting rules (if configured)

---

## Useful Commands

| Task | Command |
|------|---------|
| Check Loki logs | `kubectl -n monitoring logs loki-0 -c loki` |
| Check Promtail logs | `kubectl -n monitoring logs daemonset/promtail` |
| Check Loki is ready | `kubectl -n monitoring rollout status statefulset/loki` |
| List log buckets in RustFS | `AWS_ACCESS_KEY_ID=rustfsadmin AWS_SECRET_ACCESS_KEY=rustfsadmin aws s3 ls s3://loki --endpoint-url http://192.168.1.246:9000` |
| Upgrade Loki | `helm upgrade loki grafana/loki -n monitoring -f loki-values.yaml` |
| Upgrade Promtail | `helm upgrade promtail grafana/promtail -n monitoring` |
| Uninstall | `helm uninstall loki promtail -n monitoring` |
