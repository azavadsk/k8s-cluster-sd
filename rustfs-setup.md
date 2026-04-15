# RustFS Object Storage — Installation Guide

## Overview

RustFS is an S3-compatible object storage server written in Rust. It is deployed as a standalone service on a dedicated VM outside the Kubernetes cluster.

| Property | Value |
|----------|-------|
| Version | 1.0.0-alpha.93 |
| Host | storage (192.168.1.246) |
| OS | Rocky Linux 10.1 (Red Quartz) |
| S3 API port | 9000 |
| Console (Web UI) port | 9001 |
| Metrics port | 9090 |
| Data directory | /data/rustfs0 |
| Binary | /usr/local/bin/rustfs |
| Run as | rustfs-user |

---

## Prerequisites

- Rocky Linux 10.1 VM
- Minimum 2 CPU, 4GB RAM
- Dedicated disk or partition mounted at `/data`

---

## Step 1 — Download the Binary

Download the latest RustFS binary from the official releases:

```bash
curl -Lo /tmp/rustfs https://github.com/rustfs/rustfs/releases/download/v1.0.0-alpha.93/rustfs-linux-x86_64
sudo mv /tmp/rustfs /usr/local/bin/rustfs
sudo chmod +x /usr/local/bin/rustfs
```

Verify:

```bash
rustfs --version
```

---

## Step 2 — Create a Dedicated User

```bash
sudo useradd -r -s /sbin/nologin rustfs-user
```

---

## Step 3 — Create the Data Directory

```bash
sudo mkdir -p /data/rustfs0
sudo chown -R rustfs-user:rustfs-user /data
```

---

## Step 4 — Create the Environment Config

```bash
sudo tee /etc/default/rustfs <<EOF
RUSTFS_VOLUMES=/data/rustfs0
RUSTFS_LISTEN=0.0.0.0:9000
RUSTFS_CONSOLE_ADDR=0.0.0.0:9001
RUSTFS_ROOT_USER=rustfsadmin
RUSTFS_ROOT_PASSWORD=rustfsadmin
RUSTFS_LOG_LEVEL=info
EOF
```

**Environment variables explained:**

| Variable | Value | Description |
|----------|-------|-------------|
| `RUSTFS_VOLUMES` | `/data/rustfs0` | Directory where objects are stored |
| `RUSTFS_LISTEN` | `0.0.0.0:9000` | S3 API listen address |
| `RUSTFS_CONSOLE_ADDR` | `0.0.0.0:9001` | Web console listen address |
| `RUSTFS_ROOT_USER` | `rustfsadmin` | Admin username |
| `RUSTFS_ROOT_PASSWORD` | `rustfsadmin` | Admin password — change in production |
| `RUSTFS_LOG_LEVEL` | `info` | Log verbosity |

> **Security:** Change `RUSTFS_ROOT_USER` and `RUSTFS_ROOT_PASSWORD` before exposing to a network.

---

## Step 5 — Create the systemd Service

```bash
sudo tee /etc/systemd/system/rustfs.service <<EOF
[Unit]
Description=RustFS Object Storage Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=rustfs-user
Group=rustfs-user
WorkingDirectory=/data
EnvironmentFile=/etc/default/rustfs
ExecStart=/usr/local/bin/rustfs server
Restart=on-failure
RestartSec=10
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rustfs
```

---

## Step 6 — Disable Firewall (or Open Ports)

Disable firewall:

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

Or open only the required ports:

```bash
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --permanent --add-port=9001/tcp
sudo firewall-cmd --reload
```

---

## Step 7 — Verify

Check the service is running:

```bash
sudo systemctl status rustfs
```

Check listening ports:

```bash
ss -tlnp | grep -E '9000|9001|9090'
```

Expected:

```
LISTEN   *:9000   (S3 API)
LISTEN   *:9001   (Web console)
LISTEN   *:9090   (Metrics)
```

---

## Access

| Interface | URL | Credentials |
|-----------|-----|-------------|
| Web Console | http://192.168.1.246:9001 | rustfsadmin / rustfsadmin |
| S3 API | http://192.168.1.246:9000 | rustfsadmin / rustfsadmin |
| Metrics | http://192.168.1.246:9090 | — |

---

## Disk Layout

```
/data/
└── rustfs0/         ← object storage volume
    ├── .rustfs.sys/ ← internal metadata
    └── <buckets>/   ← user buckets
```

Storage usage:

```bash
df -h /data
du -sh /data/rustfs0
```

---

## Using the S3 API

RustFS is S3-compatible. You can use any S3 client (AWS CLI, mc, boto3).

### AWS CLI example

Configure:

```bash
aws configure set aws_access_key_id rustfsadmin
aws configure set aws_secret_access_key rustfsadmin
```

List buckets:

```bash
aws s3 ls --endpoint-url http://192.168.1.246:9000
```

Create a bucket:

```bash
aws s3 mb s3://my-bucket --endpoint-url http://192.168.1.246:9000
```

Upload a file:

```bash
aws s3 cp myfile.txt s3://my-bucket/ --endpoint-url http://192.168.1.246:9000
```

### MinIO Client (mc) example

```bash
mc alias set rustfs http://192.168.1.246:9000 rustfsadmin rustfsadmin
mc ls rustfs
mc mb rustfs/my-bucket
```

---

## Useful Commands

| Task | Command |
|------|---------|
| Check service status | `sudo systemctl status rustfs` |
| Restart service | `sudo systemctl restart rustfs` |
| View logs | `sudo journalctl -u rustfs -f` |
| View last 100 log lines | `sudo journalctl -u rustfs -n 100` |
| Check disk usage | `df -h /data` |
| Edit config | `sudo nano /etc/default/rustfs && sudo systemctl restart rustfs` |
