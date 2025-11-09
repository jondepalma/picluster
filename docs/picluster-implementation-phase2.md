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

## Phase 2: Networking & Security (Day 1-2)

### 2.1 Configure VLANs (Optional but Recommended)
**Duration:** 1 hour

If your switch supports VLANs:
- **VLAN 1:** Management (default)
- **VLAN 10:** Development environment
- **VLAN 20:** Production environment

**Note:** Can be skipped initially and added later. Use Kubernetes NetworkPolicies instead.

### 2.2 Install Tailscale VPN
**Duration:** 30 minutes

**On pi-control:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=10.0.0.0/24
```

**On your Windows machine:**
- Install Tailscale
- Connect to your tailnet
- Accept subnet routes in Tailscale admin

**Verify:**
```bash
ping <tailscale-ip-of-pi-control>
```

### 2.3 Setup Cloudflare Tunnel
**Duration:** 45 minutes

**Install cloudflared on pi-control:**
```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo dpkg -i cloudflared-linux-arm64.deb

# Authenticate
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create pi-cluster

# Get tunnel ID
cloudflared tunnel list
```

**Create tunnel configuration:**
```bash
sudo mkdir -p /etc/cloudflared

cat <<EOF | sudo tee /etc/cloudflared/config.yml
tunnel: <your-tunnel-id>
credentials-file: /root/.cloudflared/<your-tunnel-id>.json

ingress:
  - hostname: yourdomain.com
    service: http://traefik.traefik.svc.cluster.local:80
  - hostname: blog.yourdomain.com
    service: http://traefik.traefik.svc.cluster.local:80
  - hostname: api.yourdomain.com
    service: http://traefik.traefik.svc.cluster.local:80
  - service: http_status:404
EOF

# Install as service
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

**Create DNS records in Cloudflare:**
- `yourdomain.com` → CNAME → `<tunnel-id>.cfargotunnel.com`
- `blog.yourdomain.com` → CNAME → `<tunnel-id>.cfargotunnel.com`
- `api.yourdomain.com` → CNAME → `<tunnel-id>.cfargotunnel.com`
- `*.yourdomain.com` → CNAME → `<tunnel-id>.cfargotunnel.com` (wildcard)

### 2.4 Create Namespaces
**Duration:** 5 minutes

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl create namespace databases
kubectl create namespace monitoring
```

**✅ Phase 2 Complete:** Networking and security configured

---