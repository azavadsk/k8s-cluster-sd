# Trivy Security Scanning — Jenkins Pipeline

Trivy is an open-source vulnerability scanner by Aqua Security. It runs as a sidecar container inside the Jenkins agent pod and scans images **before** they are packaged with Zarf.

If vulnerabilities are found at the configured severity level, the build fails — nothing gets packaged, uploaded to RustFS, or deployed.

---

## How it works in the pipeline

```
Stage: Scan Image — Trivy
  ├── trivy image --download-db-only       (update CVE database)
  ├── trivy image --format table ...       (print human-readable report)
  ├── trivy image --format json  ...       (save JSON report as build artifact)
  └── trivy image --exit-code 1 ...        (fail build if CVEs found at threshold)
        │
        ▼ (pass only)
Stage: Build Zarf Package
Stage: Upload to RustFS
Stage: Deploy + Update ArgoCD
```

---

## Pipeline Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `IMAGE_TO_SCAN` | `nginx:alpine` | Image to scan before packaging |
| `TRIVY_SEVERITY` | `CRITICAL` | Fail if CVEs of this level found: `CRITICAL`, `CRITICAL,HIGH`, `CRITICAL,HIGH,MEDIUM` |
| `TRIVY_FAIL_ON_VULN` | `true` | Set to `false` to report only without blocking the build |

---

## Scan Report Artifacts

The pipeline saves two files as Jenkins build artifacts after every scan (pass or fail):

- `trivy-report.txt` — human-readable table for review
- `trivy-report.json` — machine-readable JSON

Access them in Jenkins under **Build → Artifacts**.

---

## Severity Levels

| Level | Meaning | Recommended action |
|-------|---------|-------------------|
| CRITICAL | Actively exploitable, severe impact | Block build |
| HIGH | Significant risk | Block build |
| MEDIUM | Moderate risk | Warn, review manually |
| LOW | Minor risk | Track, do not block |

For PSA compliance the threshold should be **CRITICAL** at minimum.

---

## Ignoring Known False Positives

Add a `.trivyignore` file to the root of your app repo to skip specific CVEs:

```
# .trivyignore
CVE-2023-12345
CVE-2023-67890
```

Trivy skips these entries in all future scans for that repo.

---

## Skipping the Scan

Set `DEPLOY_ONLY=true` to skip the scan stage and redeploy an already-built package directly from RustFS. The image was already scanned when the package was originally built.
