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

```bash
# Create Tailscale secret (get auth key from https://login.tailscale.com/admin/settings/keys)
kubectl create secret generic tailscale-auth \
  --from-literal=TS_AUTHKEY=tskey-auth-xxxxx \
  -n kube-system

# Deploy subnet router
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: tailscale-subnet-router
  namespace: kube-system
data:
  TS_ROUTES: "10.0.0.0/24"
  TS_ACCEPT_DNS: "true"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tailscale-subnet-router
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tailscale-subnet-router
  template:
    metadata:
      labels:
        app: tailscale-subnet-router
    spec:
      containers:
      - name: tailscale
        image: tailscale/tailscale:latest
        env:
        - name: TS_AUTHKEY
          valueFrom:
            secretKeyRef:
              name: tailscale-auth
              key: TS_AUTHKEY
        - name: TS_ROUTES
          valueFrom:
            configMapKeyRef:
              name: tailscale-subnet-router
              key: TS_ROUTES
        - name: TS_STATE_DIR
          value: /var/lib/tailscale
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        volumeMounts:
        - name: tailscale-state
          mountPath: /var/lib/tailscale
      volumes:
      - name: tailscale-state
        emptyDir: {}
EOF
```

**Verify:**
```bash
kubectl get pods -n kube-system -l app=tailscale-subnet-router
```

**In Tailscale admin console:**
- Approve the subnet route for 10.0.0.0/24

### 2.3 Setup Cloudflare Tunnel
**Duration:** 30 minutes

**Create tunnel via Cloudflare Dashboard:**
1. Go to https://one.dash.cloudflare.com/
2. Navigate to Access → Tunnels → Create a tunnel
3. Name: `pi-cluster-tunnel`
4. Copy the tunnel token (starts with `eyJ...`)

**Deploy to Kubernetes:**
```bash
# Create namespace and secret
kubectl create namespace cloudflare

kubectl create secret generic cloudflare-tunnel-token \
  --from-literal=token= \
  -n cloudflare

# Deploy cloudflared
kubectl apply -f ~/pi-cluster-infrastructure/networking/cloudflare/cloudflared-deployment.yaml

# Verify
kubectl get pods -n cloudflare
kubectl logs -n cloudflare -l app=cloudflared
```

**Configure public hostnames in Cloudflare Dashboard:**
- Access → Tunnels → Your tunnel → Public Hostname
- Add hostnames pointing to `traefik.traefik.svc.cluster.local:80`

**Note:** Configuration is managed via Cloudflare dashboard, not config files.

**Check Cloudflare tunnel:**
```bash
kubectl get pods -n cloudflare
kubectl logs -n cloudflare -l app=cloudflared
kubectl describe pod -n cloudflare -l app=cloudflared
```

**Deploy nginx test**

```bash
kubectl create namespace test

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-nginx
  template:
    metadata:
      labels:
        app: test-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: test-nginx
  namespace: test
spec:
  selector:
    app: test-nginx
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-nginx
  namespace: test
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: test.yourdomain.com  # Change to your actual domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-nginx
            port:
              number: 80
EOF
```

Add the test subdomain in Cloudflare:

Cloudflare Zero Trust → Tunnels → Your tunnel → Public Hostname
Add:

Subdomain: test
URL: traefik.traefik.svc.cluster.local:80
No TLS Verify: ON

**Test it:**

```bash
curl https://test.yourdomain.com
# Should see nginx welcome page HTML
```

### 2.4 Create Namespaces
**Duration:** 5 minutes

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl create namespace databases
kubectl create namespace monitoring
```

**✅ Phase 2 Complete:** Networking and security configured

## Moved some Development items into Phase 2 - PostgreSQL and Gitea

### 3.1 Deploy Development PostgreSQL
**Duration:** 30 minutes

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install postgresql-dev bitnami/postgresql \
  --namespace dev \
  --set auth.existingSecret=postgresql-dev \
  --set auth.database=devdb \
  --set primary.persistence.storageClass=local-path-nvme \
  --set primary.persistence.size=30Gi \
  --set primary.resources.requests.memory=512Mi \
  --set primary.resources.requests.cpu=250m \
  --set primary.resources.limits.memory=1Gi \
  --set primary.resources.limits.cpu=500m \
  --set primary.nodeSelector."kubernetes\.io/hostname"=pi-control \
  --set primary.tolerations[0].key=CriticalAddonsOnly \
  --set primary.tolerations[0].operator=Exists \
  --set primary.tolerations[0].effect=NoExecute
```

**Note:** Development databases use the high-performance NVMe storage on pi-control for better performance.

**Get connection details:**
```bash
export POSTGRES_PASSWORD=$(kubectl get secret --namespace dev postgresql-dev -o jsonpath="{.data.postgres-password}" | base64 -d)
echo "Connection: postgresql://postgres:$POSTGRES_PASSWORD@postgresql-dev.dev.svc.cluster.local:5432/devdb"
```

### Gitea Deployment

Reference the complete [Gitea Deployment Guide](gitea-deployment-complete-guide.md)

---