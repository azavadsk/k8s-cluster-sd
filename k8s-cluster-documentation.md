# Kubernetes Cluster Documentation

## Table of Contents

1. [Cluster Overview](#cluster-overview)
2. [Infrastructure](#infrastructure)
3. [Cluster Installation](#cluster-installation)
4. [Networking](#networking)
5. [Storage](#storage)
6. [Installed Services](#installed-services)
   - [Traefik](#traefik)
   - [MetalLB](#metallb)
   - [ArgoCD](#argocd)
   - [Kube Prometheus Stack](#kube-prometheus-stack-grafana--prometheus)
   - [Jenkins](#jenkins)
7. [DNS Setup](#dns-setup)
8. [Access Reference](#access-reference)

---

## Cluster Overview

| Property | Value |
|----------|-------|
| Kubernetes version | v1.34.6 |
| Container runtime | containerd v1.7.15 |
| OS | Rocky Linux 10.1 (Red Quartz) |
| Kernel | 6.12.0-124.8.1.el10_1.x86_64 |
| CNI | Calico (CrossSubnet mode) |
| Ingress controller | Traefik v3.6.12 |
| Load balancer | MetalLB v0.14.9 |
| Storage | local-path-provisioner |

---

## Infrastructure

### Nodes

| Node | Role | IP | OS |
|------|------|----|----|
| master | control-plane | 192.168.0.43 | Rocky Linux 10.1 |
| worker1 | worker | 192.168.1.233 | Rocky Linux 10.1 |
| worker2 | worker | 192.168.1.235 | Rocky Linux 10.1 |

### Network

| Purpose | Subnet / IP |
|---------|-------------|
| Master node | 192.168.0.x |
| Worker nodes | 192.168.1.x |
| Pod CIDR | 10.244.0.0/16 |
| MetalLB IP pool | 192.168.1.200 – 192.168.1.210 |
| Traefik external IP | 192.168.1.200 |

---

## Cluster Installation

### Prerequisites (all nodes)

Disable swap and enable required kernel modules:

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Disable firewall on all nodes:

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

### Install containerd (all nodes)

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y containerd.io
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl enable --now containerd
```

### Install kubeadm, kubelet, kubectl (all nodes)

```bash
sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

### Initialize the cluster (master only)

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.0.43

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Join worker nodes

Run the `kubeadm join` command printed by `kubeadm init` on each worker node:

```bash
sudo kubeadm join 192.168.0.43:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Install Calico CNI (master only)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml

kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - cidr: 10.244.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
EOF
```

> **Note:** `CrossSubnet` mode was used to resolve pod CIDR overlap with node IPs. If your pod network overlaps with node IPs, set `encapsulation: VXLANCrossSubnet`.

---

## Networking

### Traefik — Ingress Controller

Traefik is deployed via Helm and acts as the single entry point for all HTTP/HTTPS traffic. It is exposed via MetalLB as a `LoadBalancer` service on IP `192.168.1.200`.

All services are accessed via Traefik on:
- HTTP: `http://<hostname>` → port 80
- HTTPS: `https://<hostname>` → port 443

### MetalLB — Bare-Metal Load Balancer

MetalLB provides `LoadBalancer` service type support for bare-metal clusters using L2 (ARP) mode. It assigns IPs from a configured pool to services of type `LoadBalancer`.

**IP Pool:** `192.168.1.200 – 192.168.1.210`

Traffic flow:

```
Client request
      │
      ▼
192.168.1.200 (MetalLB virtual IP — follows Traefik pod)
      │
      ▼
Traefik pod (reads Host header, routes by Ingress rules)
      │
      ▼
Backend service pod
```

---

## Storage

### local-path-provisioner

Rancher's local-path-provisioner is used as the default StorageClass. It dynamically provisions PersistentVolumes by creating directories on the node where the pod is scheduled.

**StorageClass name:** `local-path` (default)

**Storage location on node:** `/opt/local-path-provisioner/`

Install:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml

kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

> **Limitation:** Local path volumes are tied to the node. If the node goes down, the pod will not reschedule until the node recovers. For node-failure-tolerant storage consider Longhorn or Rook/Ceph.

### Persistent Volumes

| Claim | Namespace | Size | StorageClass | Used by |
|-------|-----------|------|--------------|---------|
| jenkins | jenkins | 10Gi | local-path | Jenkins home directory |

---

## Installed Services

### Traefik

**Namespace:** `traefik`
**Helm chart:** `traefik/traefik` v39.0.7
**App version:** v3.6.12

Install:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update traefik
helm install traefik traefik/traefik --namespace traefik --create-namespace
```

After MetalLB is installed, switch from NodePort to LoadBalancer:

```bash
kubectl -n traefik patch svc traefik -p '{"spec":{"type":"LoadBalancer"}}'
```

---

### MetalLB

**Namespace:** `metallb-system`
**Version:** v0.14.9

Install:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
kubectl -n metallb-system rollout status deployment/controller --timeout=60s
```

Configure IP pool (`metallb-pool.yaml`):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: local-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: local-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - local-pool
```

```bash
kubectl apply -f metallb-pool.yaml
```

---

### ArgoCD

**Namespace:** `argocd`
**Version:** v3.3.6
**URL:** https://argocd.local

Install:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Patch argocd-server to run in insecure mode (Traefik handles TLS termination):

```bash
kubectl -n argocd patch deployment argocd-server \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/command","value":["/usr/local/bin/argocd-server","--insecure"]}]'
```

Create Ingress (`argocd-ingress.yaml`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  rules:
  - host: argocd.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
  tls:
  - hosts:
    - argocd.local
    secretName: argocd-tls
```

Get initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

> After first login, change the password in **User Info → Update Password**, then delete the secret:
> ```bash
> kubectl -n argocd delete secret argocd-initial-admin-secret
> ```

---

### Kube Prometheus Stack (Grafana + Prometheus)

**Namespace:** `monitoring`
**Helm chart:** `prometheus-community/kube-prometheus-stack` v83.0.0
**App version:** v0.90.1
**Grafana URL:** https://grafana.local

Install:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update prometheus-community

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

Grafana is exposed via Ingress at `https://grafana.local`. Default credentials: `admin` / `prom-operator`.

---

### Jenkins

**Namespace:** `jenkins`
**Helm chart:** `jenkins/jenkins` v5.9.14
**App version:** 2.541.3
**URL:** http://jenkins.local
**Persistence:** 10Gi PVC on `local-path` StorageClass

Install:

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update jenkins

kubectl create namespace jenkins

helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --set controller.serviceType=ClusterIP \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set persistence.storageClass=local-path \
  --set controller.ingress.enabled=true \
  --set controller.ingress.ingressClassName=traefik \
  --set controller.ingress.hostName=jenkins.local \
  --set controller.resources.requests.cpu=500m \
  --set controller.resources.requests.memory=512Mi \
  --set controller.resources.limits.cpu="2" \
  --set controller.resources.limits.memory=2Gi
```

Get admin password:

```bash
kubectl exec --namespace jenkins svc/jenkins -c jenkins \
  -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```

Jenkins runs as a `StatefulSet` to ensure the PVC remains attached across pod restarts. All jobs, configs, credentials, and build history are stored on the persistent volume.

---

## DNS Setup

A `dnsmasq` DNS server runs on the master node (`192.168.0.43`) and resolves all service hostnames to the MetalLB Traefik IP (`192.168.1.200`).

### Install dnsmasq (on master node)

```bash
sudo dnf install -y dnsmasq

sudo tee /etc/dnsmasq.d/k8s.conf <<EOF
address=/jenkins.local/192.168.1.200
address=/argocd.local/192.168.1.200
address=/grafana.local/192.168.1.200
EOF

sudo systemctl enable --now dnsmasq
```

### Add new service

For every new Ingress hostname, add one line and reload:

```bash
echo "address=/myapp.local/192.168.1.200" | sudo tee -a /etc/dnsmasq.d/k8s.conf
sudo systemctl reload dnsmasq
```

### Configure client machines to use this DNS

**macOS:** System Settings → Network → interface → Details → DNS → add `192.168.0.43`

**Linux:**
```bash
echo "nameserver 192.168.0.43" | sudo tee /etc/resolv.conf
```

---

## Access Reference

| Service | URL | Username | Notes |
|---------|-----|----------|-------|
| Jenkins | http://jenkins.local | admin | Password from `argocd-initial-admin-secret` |
| ArgoCD | https://argocd.local | admin | Password from `argocd-initial-admin-secret` |
| Grafana | https://grafana.local | admin | Default: `prom-operator` |

### Useful Commands

```bash
# Check all nodes
kubectl get nodes -o wide

# Check all ingresses
kubectl get ingress -A

# Check all Helm releases
helm list -A

# Check all persistent volumes
kubectl get pv,pvc -A

# Check MetalLB IP assignments
kubectl get svc -A | grep LoadBalancer

# Reload DNS after adding a service
ssh user@192.168.0.43 "sudo systemctl reload dnsmasq"
```
