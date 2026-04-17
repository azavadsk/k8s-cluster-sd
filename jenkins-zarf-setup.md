# Jenkins + Zarf CI/CD Setup

This guide configures Jenkins to automatically build Zarf packages, store them in RustFS, push images into the cluster's internal registry, and update the ArgoCD GitOps repo so ArgoCD reconciles the final state.

---

## Full Pipeline Flow

```
Git push to app repo
        │
        ▼
Jenkins Pipeline
  ├── 1. Checkout app repo (zarf.yaml + manifests)
  ├── 2. zarf package create ──► zarf-package-<name>-amd64-<ver>.tar.zst
  ├── 3. aws s3 cp           ──► s3://zarf-packages/ (RustFS)
  ├── 4. aws s3 cp (pull)    ◄── s3://zarf-packages/
  ├── 5. zarf package deploy ──► images → internal Zarf registry
  └── 6. git push            ──► azavadsk/argocd (values.yaml image tag)
                                          │
                                          ▼
                                    ArgoCD detects change
                                    reconciles cluster state
                                          │
                                          ▼
                                  Zarf mutating webhook
                                  rewrites image → internal registry
```

**Why this approach:**
- Zarf handles airgapped image delivery (images never pulled from internet at deploy time)
- ArgoCD owns the desired state — every deployment is a Git commit, fully auditable
- Zarf's mutating webhook transparently rewrites image refs on pod creation, so ArgoCD manifests stay clean with original image names

---

## Prerequisites

- Jenkins running in the cluster (namespace `jenkins`)
- ArgoCD installed and watching `github.com/azavadsk/argocd`
- `zarf init` already run on the cluster (internal registry active)
- RustFS running at `192.168.1.246:9000`

---

## Step 1 — Apply Jenkins RBAC

Jenkins agents need cluster-wide permissions to run `zarf package deploy`.

```bash
kubectl apply -f jenkins-zarf-rbac.yaml
```

Verify:

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:jenkins:jenkins -n zarf-demo
# → yes
```

---

## Step 2 — Add GitHub SSH Key to Jenkins

Jenkins needs to push commits to `azavadsk/argocd`. Use the same SSH key that has write access to that repo.

1. Jenkins → **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

| Field | Value |
|-------|-------|
| Kind | SSH Username with private key |
| ID | `github-ssh-key` |
| Username | `git` |
| Private Key | Paste the private key for `azavadsk` GitHub account |
| Description | GitHub SSH key for ArgoCD repo |

The corresponding public key must be in **GitHub → Settings → SSH keys** for the `azavadsk` account.

---

## Step 3 — Add RustFS Credentials

Jenkins → **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**:

| Field | Value |
|-------|-------|
| Kind | Username with password |
| ID | `rustfs-credentials` |
| Username | `rustfsadmin` |
| Password | `rustfsadmin` |

---

## Step 4 — Configure Global Environment Variables

Jenkins → **Manage Jenkins → System → Global properties → Environment variables**:

| Variable | Value |
|----------|-------|
| `ZARF_VERSION` | `v0.75.0` |
| `RUSTFS_URL` | `http://192.168.1.246:9000` |
| `RUSTFS_BUCKET` | `zarf-packages` |
| `ARGOCD_REPO` | `git@github.com:azavadsk/argocd.git` |
| `ARGOCD_REPO_BRANCH` | `main` |

---

## Step 5 — Create a Pipeline Job

1. Jenkins → **New Item → Pipeline** → name it e.g. `zarf-build-deploy-nginx`
2. Under **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: URL of this repo
   - Script Path: `jenkins/Jenkinsfile.zarf-build-deploy`
3. Save — parameters are defined in the Jenkinsfile and appear on first run

---

## Step 6 — Run the Pipeline

Click **Build with Parameters**:

| Parameter | Example | Description |
|-----------|---------|-------------|
| `SERVICE_NAME` | `demo-nginx` | Must match `zarf.yaml` name and ArgoCD repo folder |
| `SERVICE_VERSION` | `1.0.0` | Used as image tag in ArgoCD `values.yaml` |
| `IMAGE_REPOSITORY` | `nginx` | Image name written to `values.yaml` |
| `GIT_REPO_URL` | `https://github.com/azavadsk/k8s-cluster-sd.git` | App repo with `zarf.yaml` |
| `GIT_BRANCH` | `main` | Branch to build from |
| `DEPLOY_AFTER_BUILD` | `true` | Push images to internal registry |
| `DEPLOY_ONLY` | `false` | Skip build, redeploy from RustFS |
| `UPDATE_ARGOCD` | `true` | Push updated `values.yaml` to ArgoCD repo |

---

## Pipeline Stages Explained

| Stage | What it does |
|-------|-------------|
| **Install Tools** | Installs Zarf, AWS CLI, kubectl in the agent pod |
| **Checkout App Repo** | Clones the repo containing `zarf.yaml` and manifests |
| **Build Zarf Package** | `zarf package create` — bundles image layers + manifests into `.tar.zst` |
| **Upload to RustFS** | Stores the package in `s3://zarf-packages/` |
| **Pull from RustFS** | Downloads the package (validates storage, simulates airgap hand-off) |
| **Push Images to Internal Registry** | `zarf package deploy` — pushes image layers into the cluster's Zarf registry |
| **Update ArgoCD Repo** | Clones `azavadsk/argocd`, updates `<service>/values.yaml` with new image tag, commits and pushes |
| **Verify Deployment** | Waits 15s then checks pod status |

---

## ArgoCD Repo Structure

Each service needs a folder in `azavadsk/argocd` with a `values.yaml`. Jenkins creates this automatically on first run.

```
azavadsk/argocd/
├── demo-nginx/
│   └── values.yaml        ← Jenkins updates image.tag here
├── my-app/
│   ├── Chart.yaml
│   ├── templates/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── values.yaml
└── deployment.yaml        ← test-app raw manifest
```

The `values.yaml` Jenkins writes:

```yaml
replicaCount: 1
image:
  repository: nginx          # IMAGE_REPOSITORY parameter
  tag: "1.0.0"               # SERVICE_VERSION parameter
service:
  port: 80
```

ArgoCD reads this and applies the Deployment. The Zarf mutating webhook intercepts pod creation and rewrites the image URL to point at the internal Zarf registry.

---

## Adding a New Service

1. Create a folder `<service-name>/` in `azavadsk/argocd` with a Helm chart (copy from `my-app/` as template)
2. Create an ArgoCD `Application` pointing at that folder
3. Create a Jenkins job using `Jenkinsfile.zarf-build-deploy` with `SERVICE_NAME` matching the folder name
4. Run the pipeline — it handles the rest

---

## Triggering Automatically on Git Push

In the Jenkins job → **Build Triggers**:

- **GitHub hook trigger for GITScm polling** (requires GitHub webhook)
- Or **Poll SCM**: `H/5 * * * *`

GitHub webhook URL: `http://jenkins.sd-dev.edvantis.com/github-webhook/`

---

## Troubleshooting

### SSH push to ArgoCD repo fails

```
Permission denied (publickey)
```

Check the `github-ssh-key` credential ID matches exactly. Verify the public key is in GitHub → Settings → SSH keys for the `azavadsk` account.

### zarf deploy fails — permission denied

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:jenkins:jenkins -n zarf-demo
```

If `no` — re-apply `jenkins-zarf-rbac.yaml`.

### ArgoCD not reconciling after push

Check ArgoCD is set to auto-sync:

```bash
kubectl get application -n argocd
```

If sync policy is manual, either enable auto-sync in the ArgoCD Application spec or trigger manually from the ArgoCD UI.

### Image not found after deploy

The Zarf internal registry must have the image before ArgoCD creates the pod. If `DEPLOY_AFTER_BUILD=false` and `UPDATE_ARGOCD=true`, ArgoCD will try to create pods with an image that doesn't exist in the registry yet. Always run with `DEPLOY_AFTER_BUILD=true` on first deploy of a new version.
