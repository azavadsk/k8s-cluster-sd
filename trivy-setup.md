# Trivy Security Scanning Setup

Trivy is an open-source vulnerability scanner by Aqua Security. It is used in two places:

1. **Jenkins pipeline** — scans images before packaging (blocks bad images from being shipped)
2. **Trivy Operator in cluster** — continuously scans all running workloads and reports CVEs as Kubernetes resources

---

## Architecture

```
Jenkins Pipeline                         Kubernetes Cluster
──────────────────                       ──────────────────
Checkout code                            Trivy Operator (trivy-system)
    │                                          │
    ▼                                          │ watches all namespaces
trivy image scan ──► pass/fail                 │
    │                                          ▼
    ▼ (on pass)                         VulnerabilityReport CRDs
zarf package create                     ConfigAuditReport CRDs
    │
    ▼
deploy + ArgoCD update
```

---

## Part 1 — Trivy Operator (Cluster-Wide, Always-On)

### Installation

```bash
helm repo add aquasecurity https://aquasecurity.github.io/helm-charts
helm repo update aquasecurity

# Label namespace so Zarf agent does not rewrite Trivy image URLs
kubectl create namespace trivy-system
kubectl label namespace trivy-system zarf.dev/agent=ignore

helm install trivy-operator aquasecurity/trivy-operator \
  --namespace trivy-system \
  --set trivy.ignoreUnfixed=true \
  --set operator.scanJobTimeout=5m \
  --wait --timeout 5m
```

> **Note:** The `zarf.dev/agent=ignore` label is required. Without it, the Zarf mutating webhook rewrites Trivy's image URL to the internal Zarf registry (where the image does not exist), causing pull failures.

### Verify

```bash
kubectl get pods -n trivy-system
kubectl get vulnerabilityreports --all-namespaces
kubectl get configauditreports --all-namespaces
```

### View a vulnerability report

```bash
# List all reports
kubectl get vulnerabilityreports --all-namespaces -o wide

# Detailed report for a specific workload
kubectl describe vulnerabilityreport <report-name> -n <namespace>

# Count vulnerabilities per severity across all namespaces
kubectl get vulnerabilityreports --all-namespaces -o json | \
  jq '.items[].report.summary'
```

Example output:
```
{
  "criticalCount": 0,
  "highCount": 3,
  "lowCount": 12,
  "mediumCount": 8,
  "noneCount": 0,
  "unknownCount": 0
}
```

### What gets scanned automatically

| Resource | Report Type |
|----------|------------|
| All running container images | `VulnerabilityReport` |
| Kubernetes resource configs (RBAC, security contexts) | `ConfigAuditReport` |
| Exposed secrets in configs | `ExposedSecretReport` |

Scans run automatically when:
- A new pod is created
- An image tag changes
- On a schedule (default: every 24h)

---

## Part 2 — Trivy in Jenkins Pipeline (Pre-Build Scan)

Trivy runs as a sidecar container in the Jenkins agent pod. It scans the image **before** `zarf package create` — if vulnerabilities are found at the configured severity, the build fails and nothing gets packaged or deployed.

### How it works in the pipeline

```
Stage: Scan Image — Trivy
  ├── trivy image --download-db-only       (update CVE database)
  ├── trivy image --format table ...       (print human-readable report)
  ├── trivy image --format json ...        (save JSON report as build artifact)
  └── trivy image --exit-code 1 ...        (fail build if CVEs found)
```

### Pipeline parameters for Trivy

| Parameter | Default | Description |
|-----------|---------|-------------|
| `IMAGE_TO_SCAN` | `nginx:alpine` | Image to scan before packaging |
| `TRIVY_SEVERITY` | `CRITICAL` | Severity threshold — `CRITICAL`, `CRITICAL,HIGH`, or `CRITICAL,HIGH,MEDIUM` |
| `TRIVY_FAIL_ON_VULN` | `true` | Fail the build if vulnerabilities found at threshold |

### Scan report artifacts

The pipeline saves two files as Jenkins build artifacts:
- `trivy-report.txt` — human-readable table
- `trivy-report.json` — machine-readable JSON for integrations

Access them in Jenkins under **Build → Artifacts**.

### Setting `DEPLOY_ONLY=true` skips the scan

When redeploying an already-built package from RustFS, the scan stage is skipped (the image was already scanned when the package was built).

---

## Severity Levels

| Level | Meaning | Recommended action |
|-------|---------|-------------------|
| CRITICAL | Actively exploitable, severe impact | Block build immediately |
| HIGH | Significant risk, likely exploitable | Block build |
| MEDIUM | Moderate risk | Warn, review manually |
| LOW | Minor risk | Track, do not block |

For PSA compliance, builds should fail on **CRITICAL** at minimum.

---

## Ignoring False Positives

If a CVE is a known false positive or has no fix available, add it to a `.trivyignore` file in the repo root:

```
# .trivyignore
CVE-2023-12345
CVE-2023-67890
```

Trivy will skip these CVEs in future scans.

---

## Helm Chart Values Reference

```bash
# Show all available configuration options
helm show values aquasecurity/trivy-operator
```

Key settings used:

| Setting | Value | Effect |
|---------|-------|--------|
| `trivy.ignoreUnfixed` | `true` | Only report CVEs that have a fix available |
| `operator.scanJobTimeout` | `5m` | Timeout for each scan job |

---

## Uninstall

```bash
helm uninstall trivy-operator -n trivy-system
kubectl delete namespace trivy-system
```
