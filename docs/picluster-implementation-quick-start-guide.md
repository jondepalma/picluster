# Quick Start Guide: Raspberry Pi K3s Cluster

**Fast-track setup for Ghost blog, development environment, and production hosting**

**Time to complete:** 4-6 hours for basic setup, 8-12 hours for full configuration

---

## Prerequisites Checklist

Before starting, ensure you have:

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
sudo mount -t nfs 10.0.0.5:/share/kubernetes-pv /mnt/nas-test
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
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

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
  - 10.0.0.100-10.0.0.120
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

# Note the EXTERNAL-IP (should be in 10.0.0.100-120 range)
```

### Step 12: Install NFS Storage Provisioner

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

helm install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace kube-system \
  --set nfs.server=10.0.0.5 \
  --set nfs.path=/share/kubernetes-pv \
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
EOF

# Create directory for NVMe local-path provisioner on pi-control
ssh pi@10.0.0.10 'sudo mkdir -p /mnt/nvme/k3s-local-path-provisioner && sudo chown -R pi:pi /mnt/nvme/k3s-local-path-provisioner'
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

---

## Part 4: Networking & Access (1-2 hours)

### Step 14: Install Tailscale VPN

**On pi-control:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=10.0.0.0/24
```

**On your Windows machine:**
1. Install [Tailscale](https://tailscale.com/download/windows)
2. Sign in to your account
3. In Tailscale admin console, approve the subnet route

**Test:**
```cmd
ping <tailscale-ip-of-pi-control>
```

### Step 15: Setup Cloudflare Tunnel

**On pi-control:**

1. **Install cloudflared:**
```bash
cd ~
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo dpkg -i cloudflared-linux-arm64.deb
```

2. **Login to Cloudflare:**
```bash
cloudflared tunnel login
```
(Opens browser - authorize)

3. **Create tunnel:**
```bash
cloudflared tunnel create pi-cluster
cloudflared tunnel list
# Note your tunnel ID
```

4. **Get Traefik IP:**
```bash
kubectl get svc -n traefik traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# Example output: 10.0.0.100
```

5. **Create tunnel config:**
```bash
sudo mkdir -p /etc/cloudflared

# Replace <TUNNEL-ID> and <TRAEFIK-IP>
sudo tee /etc/cloudflared/config.yml <<EOF
tunnel: <TUNNEL-ID>
credentials-file: /home/pi/.cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: yourdomain.com
    service: http://<TRAEFIK-IP>:80
  - hostname: blog.yourdomain.com
    service: http://<TRAEFIK-IP>:80
  - hostname: "*.yourdomain.com"
    service: http://<TRAEFIK-IP>:80
  - service: http_status:404
EOF
```

6. **Install as service:**
```bash
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

7. **Create DNS records in Cloudflare:**
   - Go to Cloudflare dashboard → your domain → DNS
   - Add records:
     - `yourdomain.com` → CNAME → `<TUNNEL-ID>.cfargotunnel.com`
     - `blog.yourdomain.com` → CNAME → `<TUNNEL-ID>.cfargotunnel.com`
     - `*.yourdomain.com` → CNAME → `<TUNNEL-ID>.cfargotunnel.com`

---

## Part 5: Deploy Your First Application (30 minutes)

### Step 16: Deploy Ghost Blog

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ghost
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ghost-content
  namespace: ghost
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost
  namespace: ghost
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ghost
  template:
    metadata:
      labels:
        app: ghost
    spec:
      containers:
      - name: ghost
        image: ghost:5-alpine
        ports:
        - containerPort: 2368
        env:
        - name: url
          value: "https://blog.yourdomain.com"
        - name: NODE_ENV
          value: "production"
        volumeMounts:
        - name: content
          mountPath: /var/lib/ghost/content
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: content
        persistentVolumeClaim:
          claimName: ghost-content
---
apiVersion: v1
kind: Service
metadata:
  name: ghost
  namespace: ghost
spec:
  selector:
    app: ghost
  ports:
  - port: 2368
    targetPort: 2368
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ghost
  namespace: ghost
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
spec:
  rules:
  - host: blog.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ghost
            port:
              number: 2368
EOF
```

**Wait for deployment:**
```bash
kubectl get pods -n ghost -w
# Wait until STATUS shows "Running"
```

**Access Ghost:**
- Navigate to `https://blog.yourdomain.com`
- Go to `https://blog.yourdomain.com/ghost` to setup admin account

---

## Part 6: Development Environment (1 hour)

### Step 17: Deploy Development PostgreSQL

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install postgresql-dev bitnami/postgresql \
  --namespace dev \
  --set auth.postgresPassword=devpass123 \
  --set auth.database=devdb \
  --set primary.persistence.storageClass=nfs-db \
  --set primary.persistence.size=30Gi
```

**Get connection info:**
```bash
export PGPASSWORD=$(kubectl get secret --namespace dev postgresql-dev -o jsonpath="{.data.postgres-password}" | base64 -d)
echo "Host: postgresql-dev.dev.svc.cluster.local"
echo "Port: 5432"
echo "User: postgres"
echo "Password: $PGPASSWORD"
echo "Database: devdb"
```

### Step 18: Deploy Gitea (Git Server)

```bash
helm repo add gitea-charts https://dl.gitea.io/charts/

helm install gitea gitea-charts/gitea \
  --namespace dev \
  --set persistence.enabled=true \
  --set persistence.storageClass=nfs-client \
  --set persistence.size=50Gi \
  --set gitea.admin.username=admin \
  --set gitea.admin.password=admin123 \
  --set gitea.admin.email=admin@yourdomain.com

# Create ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea
  namespace: dev
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
spec:
  rules:
  - host: git.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitea-http
            port:
              number: 3000
EOF
```

**Access Gitea:**
- URL: `https://git.yourdomain.com`
- Username: `admin`
- Password: `admin123`

### Step 19: Setup VS Code Remote Development

**On Windows:**

1. Install [VS Code](https://code.visualstudio.com/)
2. Install "Remote - SSH" extension
3. Create SSH config (`C:\Users\YourName\.ssh\config`):

```
Host pi-dev
    HostName 10.0.0.11
    User pi
    ForwardAgent yes

Host pi-control
    HostName 10.0.0.10
    User pi
```

4. In VS Code: Press F1 → "Remote-SSH: Connect to Host" → Select `pi-dev`
5. Install Python and Node.js extensions in remote VS Code

**On pi-worker-1 (via Remote-SSH):**

```bash
# Install development tools
sudo apt install -y python3-pip python3-venv nodejs npm build-essential

# Install Docker (for building images)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker pi
```

---

## Quick Validation Checklist

After completing all steps, verify everything works:

### Cluster Health
```bash
kubectl get nodes
# All nodes should be "Ready"

kubectl get pods --all-namespaces
# All pods should be "Running"

kubectl top nodes
# Should show resource usage
```

### Storage
```bash
kubectl get pvc --all-namespaces
# All PVCs should be "Bound"

kubectl get sc
# Should see: local-path (default), nfs-client, nfs-db
```

### Networking
```bash
# Test internal DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Check LoadBalancer
kubectl get svc -n traefik
# Should show EXTERNAL-IP

# Test Cloudflare tunnel
sudo systemctl status cloudflared
# Should be "active (running)"
```

### Applications
- [ ] Ghost blog loads at https://blog.yourdomain.com
- [ ] Gitea loads at https://git.yourdomain.com
- [ ] Can connect to PostgreSQL from within cluster
- [ ] Can SSH via Tailscale VPN
- [ ] Can use VS Code Remote-SSH

---

## Next Steps

1. **Secure your cluster:**
   - Change all default passwords
   - Setup proper SSL certificates with cert-manager
   - Configure network policies

2. **Deploy production services:**
   - Production PostgreSQL (HA)
   - ScyllaDB cluster
   - Redis cache
   - Your applications

3. **Setup monitoring:**
   - Prometheus + Grafana
   - Log aggregation
   - Alerting

4. **Configure backups:**
   - Automated database backups
   - Cluster state backups
   - Test restore procedures

5. **Implement CI/CD:**
   - GitHub Actions workflows
   - Automated testing
   - Deployment pipelines

---

## Common Issues & Solutions

### Issue: Node shows "NotReady"
```bash
# Check node logs
sudo journalctl -u k3s -f  # on control plane
sudo journalctl -u k3s-agent -f  # on workers

# Restart K3s
sudo systemctl restart k3s  # or k3s-agent
```

### Issue: Pod stuck in "Pending"
```bash
# Check events
kubectl describe pod <pod-name> -n <namespace>

# Check PVC
kubectl get pvc -n <namespace>
```

### Issue: NFS mount fails
```bash
# Test NFS manually
sudo mount -t nfs 10.0.0.5:/share/kubernetes-pv /mnt/test
ls -la /mnt/test
sudo umount /mnt/test

# Check QNAP NFS service
# Via QNAP web interface: Network & File Services → NFS
```

### Issue: Ingress not working
```bash
# Check Traefik status
kubectl get pods -n traefik

# Check ingress
kubectl get ingress --all-namespaces
kubectl describe ingress <ingress-name> -n <namespace>

# Check Cloudflare tunnel
sudo journalctl -u cloudflared -f
```

### Issue: Can't reach services externally
```bash
# Verify Cloudflare tunnel is running
curl -I https://blog.yourdomain.com

# Check DNS records in Cloudflare dashboard
# Ensure CNAME points to your tunnel ID
```

---

## Essential Commands Reference

### Cluster Management
```bash
# View all resources
kubectl get all --all-namespaces

# View resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# View logs
kubectl logs -f <pod-name> -n <namespace>

# Exec into pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Port forward
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
```

### Application Management
```bash
# Scale deployment
kubectl scale deployment <name> --replicas=3 -n <namespace>

# Restart deployment
kubectl rollout restart deployment/<name> -n <namespace>

# Check rollout status
kubectl rollout status deployment/<name> -n <namespace>

# Update image
kubectl set image deployment/<name> <container>=<new-image> -n <namespace>
```

### Debugging
```bash
# Run debug pod
kubectl run -it --rm debug --image=busybox --restart=Never -- sh

# Check DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup <service>.<namespace>.svc.cluster.local

# Check connectivity
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl <url>
```

---

## Support & Resources

- **K3s Documentation:** https://docs.k3s.io
- **Kubernetes Documentation:** https://kubernetes.io/docs
- **Helm Charts:** https://artifacthub.io
- **Traefik Documentation:** https://doc.traefik.io/traefik
- **Cloudflare Tunnel Docs:** https://developers.cloudflare.com/cloudflare-one/connections/connect-apps

---

## Congratulations! 🎉

You now have a fully functional Kubernetes cluster running on Raspberry Pi hardware with:
- ✅ K3s lightweight Kubernetes
- ✅ Persistent storage on QNAP NAS
- ✅ Load balancing with MetalLB
- ✅ Ingress routing with Traefik
- ✅ Secure VPN access via Tailscale
- ✅ Public access via Cloudflare Tunnel
- ✅ Ghost blog running
- ✅ Development environment with Gitea and PostgreSQL
- ✅ VS Code remote development

**Your cluster is ready for application deployment!**
