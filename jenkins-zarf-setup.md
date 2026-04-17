# Jenkins + Zarf CI/CD Setup

This guide configures Jenkins to automatically build Zarf packages from a Git repository, store them in RustFS, and deploy them to the Kubernetes cluster.

---

## Architecture

```
Git Push
   │
   ▼
Jenkins Pipeline
   ├── 1. Checkout source from Git
   ├── 2. zarf package create  ──► packages/zarf-package-<name>-amd64-<ver>.tar.zst
   ├── 3. aws s3 cp            ──► s3://zarf-packages/ (RustFS)
   ├── 4. aws s3 cp (pull)     ◄── s3://zarf-packages/
   └── 5. zarf package deploy  ──► Kubernetes cluster
```

Each service has its own Git repo with a `zarf.yaml` at the root. One shared Jenkinsfile handles all services via parameters.

---

## Step 1 — Apply Jenkins RBAC

Jenkins agents run as the `jenkins` service account. They need cluster-wide permissions to create namespaces and apply manifests.

```bash
kubectl apply -f jenkins-zarf-rbac.yaml
```

Verify:

```bash
kubectl get clusterrolebinding jenkins-zarf-deploy
```

---

## Step 2 — Configure Jenkins Global Environment Variables

In Jenkins → **Manage Jenkins → System → Global properties → Environment variables**, add:

| Variable | Value |
|----------|-------|
| `ZARF_VERSION` | `v0.75.0` |
| `RUSTFS_URL` | `http://192.168.1.246:9000` |
| `RUSTFS_BUCKET` | `zarf-packages` |

---

## Step 3 — Add RustFS Credentials

In Jenkins → **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**:

| Field | Value |
|-------|-------|
| Kind | Username with password |
| ID | `rustfs-credentials` |
| Username | `rustfsadmin` |
| Password | `rustfsadmin` |
| Description | RustFS S3 credentials |

---

## Step 4 — Create a Pipeline Job

1. Jenkins → **New Item → Pipeline**
2. Name it after the service (e.g. `zarf-build-deploy-nginx`)
3. Under **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: URL of your infra/manifests repo
   - Script Path: `jenkins/Jenkinsfile.zarf-build-deploy`
4. Enable **This project is parameterized** — the Jenkinsfile defines the parameters automatically on first run

Or use **Pipeline script** and paste the contents of `Jenkinsfile.zarf-build-deploy` directly.

---

## Step 5 — Run the Pipeline

Click **Build with Parameters** and fill in:

| Parameter | Example |
|-----------|---------|
| `SERVICE_NAME` | `demo-nginx` |
| `SERVICE_VERSION` | `1.0.0` |
| `GIT_REPO_URL` | `https://github.com/azavadsk/k8s-cluster-sd.git` |
| `GIT_BRANCH` | `main` |
| `DEPLOY_AFTER_BUILD` | `true` |
| `DEPLOY_ONLY` | `false` |

---

## Pipeline Stages Explained

| Stage | What it does |
|-------|-------------|
| **Install Zarf** | Downloads the Zarf binary into the build agent pod |
| **Checkout** | Clones the Git repo containing `zarf.yaml` and manifests |
| **Build Zarf Package** | Runs `zarf package create` — bundles images + manifests into `.tar.zst` |
| **Upload to RustFS** | Pushes the package to `s3://zarf-packages/` on RustFS |
| **Pull from RustFS** | Downloads the package back (validates storage round-trip) |
| **Deploy to Cluster** | Runs `zarf package deploy` — pushes images to internal registry and applies manifests |

---

## Per-Service Repo Structure

Each service that gets packaged with Zarf must have this layout in its Git repo:

```
my-service/
├── zarf.yaml              # Package definition
├── manifests/
│   ├── deployment.yaml
│   └── service.yaml
└── Jenkinsfile            # Optional: service-specific overrides
```

### zarf.yaml template

```yaml
kind: ZarfPackageConfig
metadata:
  name: my-service          # Must match SERVICE_NAME parameter
  version: 1.0.0            # Must match SERVICE_VERSION parameter
  description: My service description

components:
  - name: my-service
    required: true
    manifests:
      - name: my-service
        namespace: my-service
        files:
          - manifests/deployment.yaml
          - manifests/service.yaml
    images:
      - my-registry/my-image:tag
```

---

## Deploy Only (Skip Build)

To redeploy an already-built package from RustFS without rebuilding:

- Set `DEPLOY_ONLY = true`
- Set `SERVICE_NAME` and `SERVICE_VERSION` to match the existing package filename in RustFS

The pipeline will skip Checkout and Build stages, pull the package directly from RustFS, and deploy.

---

## Triggering Automatically on Git Push

In the Jenkins job config:

1. **Build Triggers** → enable **GitHub hook trigger for GITScm polling** (if using GitHub webhooks)
2. Or enable **Poll SCM** with schedule `H/5 * * * *` (every 5 minutes)

For GitHub webhooks:
- Go to your GitHub repo → Settings → Webhooks → Add webhook
- Payload URL: `http://jenkins.sd-dev.edvantis.com/github-webhook/`
- Content type: `application/json`
- Events: `Just the push event`

---

## Troubleshooting

### Agent pod fails to start

```bash
kubectl get pods -n jenkins
kubectl describe pod <agent-pod> -n jenkins
```

Common cause: the `amazon/aws-cli` image can't be pulled. Either pre-load it or use a lighter base image.

### zarf deploy fails with permission denied

Check RBAC is applied:
```bash
kubectl auth can-i create deployments --as=system:serviceaccount:jenkins:jenkins -n zarf-demo
```

Should return `yes`. If not, re-apply `jenkins-zarf-rbac.yaml`.

### RustFS upload fails

Verify credentials and connectivity from a Jenkins agent:
```bash
AWS_ACCESS_KEY_ID=rustfsadmin \
AWS_SECRET_ACCESS_KEY=rustfsadmin \
aws s3 ls s3://zarf-packages/ --endpoint-url http://192.168.1.246:9000
```

### Package not found in RustFS (DEPLOY_ONLY mode)

List what's available:
```bash
AWS_ACCESS_KEY_ID=rustfsadmin \
AWS_SECRET_ACCESS_KEY=rustfsadmin \
aws s3 ls s3://zarf-packages/ --endpoint-url http://192.168.1.246:9000
```

The filename must match `zarf-package-<SERVICE_NAME>-amd64-<SERVICE_VERSION>.tar.zst`.
