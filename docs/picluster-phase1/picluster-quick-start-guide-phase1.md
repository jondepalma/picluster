# Quick Start Guide: Raspberry Pi K3s Cluster

**Fast-track setup for Ghost blog, development environment, and production hosting**

**Time to complete:** 4-6 hours for basic setup, 8-12 hours for full configuration

---

## Prerequisites Checklist

Before starting, this configuration assumes you have the following:

- [ ] 3x Raspberry Pi 5 (8GB RAM each)
- [ ] 3x MicroSD cards (256GB, A2 rated)
- [ ] 1x M.2 NVMe SSD (1TB) + M.2 HAT for pi-control
- [ ] QNAP NAS (configured with static IP)
- [ ] Gigabit network switch
- [ ] Power supplies for all Pis (27W USB-C for Pi 5)
- [ ] Domain name (for Cloudflare tunnel)
- [ ] Cloudflare account
- [ ] Tailscale account
- [ ] Windows machine with VS Code

---

## Part 1: Initial Setup (2 hours)

### Step 1: Flash Raspberry Pi OS

**On your Windows machine:**

1. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
2. Flash all 3 SD cards with:
   - OS: **Raspberry Pi OS Lite (64-bit)**
   - Settings (gear icon):
     - Enable SSH (password authentication)
     - Set username: `pi`
     - Set password: `<your-password>`
     - Configure WiFi (optional)
     - Set hostname:
       - Card 1: `pi-control`
       - Card 2: `pi-worker-1`
       - Card 3: `pi-worker-2`

3. Write to all cards

**Setup NVMe SSD on pi-control:**

1. Attach M.2 HAT to pi-control
2. Insert 1TB NVMe SSD into M.2 HAT
3. Boot pi-control first

### Step 2: First Boot and Network Configuration

1. **Insert SD cards** into Pis and power on
2. **Find IP addresses** (from your router DHCP list)
3. **SSH into each Pi:**

```bash
ssh pi@<ip-address>
```

4. **Format and mount NVMe on pi-control first:**

```bash
# On pi-control only
lsblk  # Should see nvme0n1 (1TB)

# Format NVMe
sudo mkfs.ext4 /dev/nvme0n1

# Create mount point
sudo mkdir -p /mnt/nvme

# Add to fstab for automatic mounting
echo '/dev/nvme0n1 /mnt/nvme ext4 defaults,noatime 0 2' | sudo tee -a /etc/fstab

# Mount it
sudo mount -a

# Verify
df -h | grep nvme
# Should show ~1TB mounted at /mnt/nvme

# Set permissions
sudo chown pi:pi /mnt/nvme
```

5. **Set static IPs on each node:**

```bash
# On pi-control (10.0.0.10)
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses 10.0.0.10/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "10.0.0.1 1.1.1.1" \
  ipv4.method manual

# On pi-worker-1 (10.0.0.11)
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses 10.0.0.11/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "10.0.0.1 1.1.1.1" \
  ipv4.method manual

# On pi-worker-2 (10.0.0.12)
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses 10.0.0.12/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "10.0.0.1 1.1.1.1" \
  ipv4.method manual
```

6. **Reboot all nodes:**
```bash
sudo reboot
```

7. **SSH back in using new IPs:**
```bash
ssh pi@10.0.0.10  # control plane
ssh pi@10.0.0.11  # worker 1
ssh pi@10.0.0.12  # worker 2
```

### Step 3: Update All Systems

**Run on ALL nodes (can do in parallel):**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nfs-common curl git vim htop
```

### Step 4: Configure QNAP NAS

**On your QNAP (via web interface):**

1. **Set static IP:**
   - Control Panel → Network & Virtual Switch → Interface
   - IP: `10.0.0.5`
   - Netmask: `255.255.255.0`
   - Gateway: `10.0.0.1`

2. **Enable NFS:**
   - Control Panel → Network & File Services → Win/Mac/NFS
   - Check "Enable NFS service"
   - Enable NFSv4

3. **Create shared folders:**
   - Control Panel → Shared Folders → Create
   - Name: `kubernetes-pv`
   - Size: 500GB minimum
   - Create another: `kubernetes-backups` (100GB)

4. **Configure NFS permissions:**
   - Shared Folders → kubernetes-pv → Edit Shared Folder Permissions
   - NFS Host Access:
     - Add `10.0.0.0/24`
     - Permission: Read/Write
     - Check "No root squash"
   - Repeat for kubernetes-backups

5. **Test NFS from pi-control:**
```bash
sudo mkdir -p /mnt/nas-test
sudo mount -t nfs -o vers=3 10.0.0.5:/share/CACHEDEV5_DATA/kubernetes-pv /mnt/nas-test
ls -la /mnt/nas-test
sudo umount /mnt/nas-test
```

---

## Part 2: Install K3s Cluster (1 hour)

### Step 5: Install K3s Control Plane

**On pi-control (10.0.0.10):**

```bash
# Install K3s control plane
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode 644 \
  --disable traefik \
  --disable servicelb \
  --node-taint CriticalAddonsOnly=true:NoExecute

# Wait for K3s to start
sudo systemctl status k3s

# Verify
sudo kubectl get nodes
```

If K3s do not start, verify no cgroup error
I have added the arguments for `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` commandline options to `/boot/firmware/cmdline.txt`.
I wanted a clean install so I ran `sudo /usr/local/bin/k3s-uninstall.sh`
Then issued a `sudo reboot`
Then retried the above install command, and it worked

```bash
sudo nano /boot/firmware/cmdline.txt

# **Add the following parameters to the end of the existing line** (it should all be on ONE line):

# cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory


# **Important:** 
# - Make sure everything stays on a **single line**
# - Add a space before adding these parameters
# - Don't create new lines

# The complete line should look something like this:

# console=serial0,115200 console=tty1 root=PARTUUID=xxxxx rootfstype=ext4 ... cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory

sudo reboot

```

### Step 6: Get Join Token

**On pi-control:**

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Copy this token - you'll need it for worker nodes.

### Step 7: Join Worker Nodes

**On each worker (pi-worker-1, pi-worker-2, pi-worker-3):**

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.0.10:6443 \
  K3S_TOKEN=<paste-token-here> sh -
```

**Verify on pi-control:**

```bash
sudo kubectl get nodes

# Should show:
# NAME          STATUS   ROLES                  AGE
# pi-control    Ready    control-plane,master   2m
# pi-worker-1   Ready    <none>                 1m
# pi-worker-2   Ready    <none>                 1m
# pi-worker-3   Ready    <none>                 1m
```

### Step 8: Configure kubectl on Your Windows Machine

**On pi-control, export kubeconfig:**

```bash
sudo cat /etc/rancher/k3s/k3s.yaml
```

**On Windows:**

1. Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
2. Create `C:\Users\YourName\.kube\config`
3. Paste the kubeconfig content
4. **Important:** Change the server line:
   - From: `server: https://127.0.0.1:6443`
   - To: `server: https://10.0.0.10:6443`

5. Test:
```cmd
kubectl get nodes
```

---

## Part 3: Essential Infrastructure (2 hours)

### Step 9: Install Helm

**On pi-control:**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### Step 10: Install MetalLB (Load Balancer)

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml

# Wait for pods to be ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# Configure IP address pool
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

### Step 11: Install Traefik (Ingress)

When installing Traefik, it was attempting to use local host to connect to the cluster. The Kubernetes API server config needed to be updated.

The issue is that Helm is looking for the Kubernetes API server at localhost:8080 instead of using your K3s configuration. Here's how to resolve it:

K3s stores its kubeconfig at /etc/rancher/k3s/k3s.yaml instead of the standard ~/.kube/config location. Helm (and kubectl) need to know where to find this configuration file.

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Verify it works
kubectl get nodes

```

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

kubectl create namespace traefik

helm install traefik traefik/traefik \
  --namespace traefik \
  --set service.type=LoadBalancer \
  --set ports.web.port=80 \
  --set ports.websecure.port=443

# Get the LoadBalancer IP
kubectl get svc -n traefik traefik

# Note the EXTERNAL-IP (should be in 10.0.0.30-50 range)
```

### Step 12: Install NFS Storage Provisioner

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

helm install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace kube-system \
  --set nfs.server=10.0.0.5 \
  --set nfs.path=/share/CACHEDEV5_DATA/kubernetes-pv \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=false

# Create storage classes
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
provisioner: cluster.local/nfs-provisioner-nfs-subdir-external-provisioner
reclaimPolicy: Retain
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-db
provisioner: cluster.local/nfs-provisioner-nfs-subdir-external-provisioner
reclaimPolicy: Retain
mountOptions:
  - hard
  - nfsvers=4.1
  - noatime
EOF

# Create directory for NVMe local-path provisioner on pi-control
ssh pi@10.0.0.10 'sudo mkdir -p /mnt/nvme/k3s-local-path-provisioner && sudo chown -R pi:pi /mnt/nvme/k3s-local-path-provisioner'
```
This also had an error as the nfs-client storage class was already created. I had to verify it wasn't in use, delete it, and then re-run the command, after this the storage classes were configured properly.

```bash
kubectl get pvc -A -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,STORAGECLASS:.spec.storageClassName | grep nfs-client
# If no results, then nothing is using it and it can be deleted.
```

```bash
# If nothing is using nfs-client, delete it
kubectl delete storageclass nfs-client
# Then re-run the storage class command again
# Then verify results
kubectl get storageclass
```

There was also an error with the original provisioner names, which needed to be recreated (original here, current correct in the config above)

```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: cluster.local/nfs-subdir-external-provisioner
reclaimPolicy: Retain
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-db
provisioner: cluster.local/nfs-subdir-external-provisioner
reclaimPolicy: Retain
mountOptions:
  - hard
  - nfsvers=4.1
  - noatime
```

**Storage Class Guide:**
- **local-path** (default): SD card storage, temporary data
- **local-path-nvme**: High-performance local storage on pi-control's NVMe
- **nfs-client**: Persistent bulk storage on NAS
- **nfs-db**: Database-optimized NFS storage

### Step 13: Create Namespaces

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl create namespace monitoring
```