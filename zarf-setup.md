# Zarf Packaging — Setup & Deployment Guide

Zarf is an airgap-first packaging tool for Kubernetes. It bundles container images, manifests, and Helm charts into a single signed `.tar.zst` file that can be deployed without any external connectivity. This aligns with the ELECTORA PSA airgapped deployment model.

---

## Architecture Overview

```
[Local Machine]                [RustFS]                   [K8s Cluster]
      |                            |                             |
  zarf build ──────────────► s3://zarf-packages/               |
  (amd64 package)                 |                             |
                            aws s3 cp ◄────────────────────────|
                                  |                             |
                            zarf init ──────────────────────── ► zarf namespace
                            zarf deploy ────────────────────── ► app namespace
```

**Key principle:** The package is built once, stored in RustFS, and can be deployed to any cluster — even one with no internet access.

---

## Prerequisites

- `zarf` binary installed (see Installation below)
- `kubectl` configured and pointing at the target cluster
- AWS CLI configured for RustFS access
- RustFS running at `192.168.1.246:9000`

---

## Installation

Zarf is not in Homebrew. Download the binary directly:

```bash
# macOS Apple Silicon (arm64)
curl -L -o ~/zarf \
  https://github.com/zarf-dev/zarf/releases/download/v0.75.0/zarf-darwin-arm64 \
  && chmod +x ~/zarf

# Move to PATH permanently
sudo mv ~/zarf /usr/local/bin/zarf

# Verify
zarf version
```

---

## Project Structure

```
SmithsDetection/zarf-demo/
├── zarf.yaml                   # Package definition
├── manifests/
│   ├── deployment.yaml         # Kubernetes Deployment
│   └── service.yaml            # Kubernetes Service
└── deploy/                     # Working dir for init + package files
    ├── zarf-init-amd64-v0.75.0.tar.zst
    └── zarf-package-demo-nginx-amd64-1.0.0.tar.zst
```

### zarf.yaml

```yaml
kind: ZarfPackageConfig
metadata:
  name: demo-nginx
  version: 1.0.0
  description: Simple nginx demo app for Zarf packaging test

components:
  - name: demo-nginx
    required: true
    manifests:
      - name: demo-nginx
        namespace: zarf-demo
        files:
          - manifests/deployment.yaml
          - manifests/service.yaml
    images:
      - nginx:alpine
```

---

## Step 1 — Build the Package

Build must use `--architecture amd64` when building on an Apple Silicon Mac for amd64 cluster nodes.

```bash
cd /Users/andrii.zavadskiy/EDVANTIS/SmithsDetection/zarf-demo

zarf package create . \
  --output ./packages \
  --architecture amd64 \
  --confirm
```

Output: `packages/zarf-package-demo-nginx-amd64-1.0.0.tar.zst` (~25 MB)

What gets bundled:
- `nginx:alpine` container image (amd64 variant)
- `manifests/deployment.yaml` and `manifests/service.yaml`
- Auto-generated SBOM (Software Bill of Materials)
- Checksums for integrity verification

---

## Step 2 — Upload to RustFS

```bash
AWS_ACCESS_KEY_ID=rustfsadmin \
AWS_SECRET_ACCESS_KEY=rustfsadmin \
aws s3 cp packages/zarf-package-demo-nginx-amd64-1.0.0.tar.zst \
  s3://zarf-packages/ \
  --endpoint-url http://192.168.1.246:9000
```

Verify it was uploaded:

```bash
AWS_ACCESS_KEY_ID=rustfsadmin \
AWS_SECRET_ACCESS_KEY=rustfsadmin \
aws s3 ls s3://zarf-packages/ \
  --endpoint-url http://192.168.1.246:9000
```

---

## Step 3 — Pull Package from RustFS (Airgap Simulation)

In a real airgapped environment this step happens on the DMZ device or operator machine that has access to the cluster. Here it runs from the local machine.

```bash
mkdir -p /Users/andrii.zavadskiy/EDVANTIS/SmithsDetection/zarf-demo/deploy

AWS_ACCESS_KEY_ID=rustfsadmin \
AWS_SECRET_ACCESS_KEY=rustfsadmin \
aws s3 cp \
  s3://zarf-packages/zarf-package-demo-nginx-amd64-1.0.0.tar.zst \
  /Users/andrii.zavadskiy/EDVANTIS/SmithsDetection/zarf-demo/deploy/ \
  --endpoint-url http://192.168.1.246:9000
```

---

## Step 4 — Bootstrap Zarf in the Cluster (One-Time)

`zarf init` installs Zarf's internal container registry and mutating webhook agent into the cluster. This is a **one-time operation per cluster**.

### Download the init package

The init package must match the cluster architecture (amd64):

```bash
curl -L -o /Users/andrii.zavadskiy/EDVANTIS/SmithsDetection/zarf-demo/deploy/zarf-init-amd64-v0.75.0.tar.zst \
  https://github.com/zarf-dev/zarf/releases/download/v0.75.0/zarf-init-amd64-v0.75.0.tar.zst
```

### Run init

```bash
cd /Users/andrii.zavadskiy/EDVANTIS/SmithsDetection/zarf-demo/deploy

zarf init ./zarf-init-amd64-v0.75.0.tar.zst --confirm
```

What `zarf init` installs:
| Component | Description |
|-----------|-------------|
| `zarf-injector` | Bootstraps the registry by cloning a running pod. Removed after seeding. |
| `zarf-seed-registry` | Temporary registry used during bootstrap |
| `zarf-registry` | Permanent internal Docker registry (20Gi PVC) |
| `zarf-agent` | Mutating webhook that rewrites image URLs to point at the internal registry |

Verify:

```bash
kubectl get pods -n zarf
```

Expected output:
```
zarf-docker-registry-...   1/1   Running
zarf-d2db14ef40305397-...  1/1   Running   # zarf-agent
```

### Optional: Add git server

```bash
zarf init ./zarf-init-amd64-v0.75.0.tar.zst --components=git-server --confirm
```

---

## Step 5 — Deploy the Package

```bash
cd /Users/andrii.zavadskiy/EDVANTIS/SmithsDetection/zarf-demo/deploy

zarf package deploy ./zarf-package-demo-nginx-amd64-1.0.0.tar.zst --confirm
```

What happens during deploy:
1. Zarf pushes the bundled `nginx:alpine` image into the internal Zarf registry
2. The Zarf agent (mutating webhook) rewrites the Deployment's image reference from `docker.io/library/nginx:alpine` to the internal registry URL
3. Kubernetes manifests are applied — Deployment and Service are created in the `zarf-demo` namespace

---

## Step 6 — Verify

```bash
kubectl get pods,svc -n zarf-demo
```

Expected output:
```
NAME                              READY   STATUS    RESTARTS   AGE
pod/demo-nginx-6fdbf8b4cf-d5bcp   1/1     Running   0          2m

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/demo-nginx   ClusterIP   10.100.225.36   <none>        80/TCP    2m
```

Test the service from inside the cluster:

```bash
kubectl run test --image=busybox --rm -it --restart=Never -n zarf-demo \
  -- wget -qO- http://demo-nginx
```

---

## RustFS Bucket Layout

| Bucket | Contents |
|--------|----------|
| `zarf-packages` | Built Zarf packages (`.tar.zst` files) |

Access RustFS Web Console: `http://192.168.1.246:9001`  
Credentials: `rustfsadmin` / `rustfsadmin`

---

## Zarf Credentials

After `zarf init`, get the internal registry credentials:

```bash
zarf tools get-creds
```

---

## Architecture Notes

| Context | Architecture |
|---------|-------------|
| Local build machine (Mac M1/M2/M3) | `arm64` |
| Cluster nodes (Rocky Linux VMs) | `amd64` |
| Package must be built as | `amd64` |
| Init package must be | `amd64` |

Always pass `--architecture amd64` when building on Apple Silicon for this cluster.

---

## Updating a Package

1. Edit `zarf.yaml` or manifests
2. Bump `version` in `zarf.yaml` (e.g. `1.1.0`)
3. Rebuild: `zarf package create . --output ./packages --architecture amd64 --confirm`
4. Upload new version to RustFS
5. Pull and deploy as above — Zarf will upgrade the existing deployment

---

## Removing a Deployment

```bash
# Remove a specific package
zarf package remove demo-nginx --confirm

# Tear down Zarf entirely (removes registry, agent, all Zarf resources)
zarf destroy --confirm
```
