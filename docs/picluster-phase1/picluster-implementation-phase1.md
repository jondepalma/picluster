# Simplified Pi Cluster Implementation Plan

**Focus:** Ghost blog, development environment, production hosting, and essential infrastructure

**Timeline:** 2-3 days for core setup, 1 week for full deployment

---

## Overview

This simplified architecture focuses on:
- **Public Website:** Ghost blog with Cloudflare tunnel
- **Development Environment:** Python/Node.js/React with local databases
- **Production Environment:** Hosting APIs and applications
- **Infrastructure:** K3s, networking, storage, and security

---

## Phase 1: Foundation (Day 1)

### 1.1 Hardware Setup
**Duration:** 2 hours

**Components:**
- 3x Raspberry Pi 5 (8GB RAM each)
  - pi-control: 256GB SD + 1TB M.2 NVMe SSD
  - pi-worker-1: 256GB SD
  - pi-worker-2: 256GB SD
- QNAP NAS (10.0.0.5)
- Gigabit switch
- Power supplies

**Tasks:**
- [x] Flash Raspberry Pi OS Lite (64-bit) to all SD cards
- [x] Attach 1TB NVMe SSD to pi-control (M.2 HAT)
- [x] Boot all nodes and complete initial setup
- [x] Configure static IPs:
  - pi-control: 10.0.0.10
  - pi-worker-1: 10.0.0.11
  - pi-worker-2: 10.0.0.12
  - QNAP NAS: 10.0.0.5
- [x] Set hostnames on each node
- [x] Enable SSH on all nodes
- [x] Update all systems: `sudo apt update && sudo apt upgrade -y`
- [x] Mount and format NVMe on pi-control:
  ```bash
  # On pi-control
  lsblk  # Identify NVMe (usually /dev/nvme0n1)
  sudo mkfs.ext4 /dev/nvme0n1
  sudo mkdir -p /mnt/nvme
  echo '/dev/nvme0n1 /mnt/nvme ext4 defaults 0 2' | sudo tee -a /etc/fstab
  sudo mount -a
  df -h  # Verify mount
  ```

### 1.2 QNAP NAS Configuration
**Duration:** 1 hour

**Tasks:**
- [x] Assign static IP: 10.0.0.5
- [x] Enable NFS service (Control Panel → Network & File Services → NFS)
- [x] Create shared folders:
  - `/kubernetes-pv` (500GB) - Persistent volumes
  - `/kubernetes-backups` (100GB) - Backups
- [x] Configure NFS permissions:
  - Allow 10.0.0.0/24 network
  - Read/Write access
  - Enable "No root squash"
- [x] Test NFS from pi-control:
  ```bash
  sudo apt install nfs-common
  sudo mkdir -p /mnt/nas-test
  sudo mount -t nfs -o vers=3 10.0.0.5:/share/CACHEDEV5_DATA/kubernetes-pv /mnt/nas-test
  ```

### 1.3 Install K3s Cluster
**Duration:** 30 minutes

**Control Plane (pi-control):**
```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode 644 \
  --disable traefik \
  --disable servicelb \
  --node-taint CriticalAddonsOnly=true:NoExecute
```

**Get Join Token:**
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

**Worker Nodes (pi-worker-1, pi-worker-2, pi-worker-3):**
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.0.10:6443 \
  K3S_TOKEN=<your-token> sh -
```

**Verify:**
```bash
kubectl get nodes
# All nodes should show Ready
```

### 1.4 Install Core Networking
**Duration:** 1 hour

**Install Helm:**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Install MetalLB (Load Balancer):**
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml

# Wait for pods
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# Configure IP pool
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.30-10.0.0.50
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF
```

**Install Traefik (Ingress Controller):**
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set service.type=LoadBalancer \
  --set ports.web.port=80 \
  --set ports.websecure.port=443 \
  --set persistence.enabled=true \
  --set persistence.size=1Gi
```

**Verify Traefik gets external IP:**
```bash
kubectl get svc -n traefik
# Should show EXTERNAL-IP in 10.0.0.30-50 range
```

### 1.5 Install cert-manager (SSL)
**Duration:** 20 minutes

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Wait for pods
kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=90s
```

### 1.6 Setup NFS Storage Provisioner
**Duration:** 30 minutes

**Install NFS client on all nodes:**
```bash
# Run on each Pi node
sudo apt install -y nfs-common
```

**Deploy NFS Provisioner:**
```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

helm install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace kube-system \
  --set nfs.server=10.0.0.5 \
  --set nfs.path=/share/CACHEDEV5_DATA/kubernetes-pv \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=false \
  --set storageClass.archiveOnDelete=true
```

**Create Storage Classes:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-nvme
  annotations:
    storageclass.kubernetes.io/description: "High-performance NVMe storage on pi-control"
provisioner: rancher.io/local-path
parameters:
  nodePath: /mnt/nvme/k3s-local-path-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: cluster.local/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "true"
reclaimPolicy: Retain
volumeBindingMode: Immediate
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-db
provisioner: cluster.local/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "false"
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1
  - noatime
  - nodiratime
EOF

# Create directory for NVMe provisioner
sudo mkdir -p /mnt/nvme/k3s-local-path-provisioner
sudo chown -R root:root /mnt/nvme/k3s-local-path-provisioner
```

**Storage Strategy:**
- **local-path**: Default, SD card storage for logs/temp data
- **local-path-nvme**: High-performance local storage on pi-control's 1TB NVMe (databases, builds)
- **nfs-client**: Bulk persistent storage on NAS (backups, repos, large files)
- **nfs-db**: Database-optimized NFS storage with performance tuning

**✅ Phase 1 Complete:** Core cluster is ready

---