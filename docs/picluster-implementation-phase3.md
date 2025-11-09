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

## Phase 3: Development Environment (Day 2)

### 3.1 Deploy Development PostgreSQL
**Duration:** 30 minutes

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install postgresql-dev bitnami/postgresql \
  --namespace dev \
  --set auth.postgresPassword=devpassword \
  --set auth.database=devdb \
  --set primary.persistence.storageClass=local-path-nvme \
  --set primary.persistence.size=30Gi \
  --set primary.resources.requests.memory=512Mi \
  --set primary.resources.requests.cpu=250m \
  --set primary.nodeSelector."kubernetes\.io/hostname"=pi-control
```

**Note:** Development databases use the high-performance NVMe storage on pi-control for better performance.

**Get connection details:**
```bash
export POSTGRES_PASSWORD=$(kubectl get secret --namespace dev postgresql-dev -o jsonpath="{.data.postgres-password}" | base64 -d)
echo "Connection: postgresql://postgres:$POSTGRES_PASSWORD@postgresql-dev.dev.svc.cluster.local:5432/devdb"
```

### 3.2 Deploy Development ScyllaDB
**Duration:** 45 minutes

**Install Scylla Operator:**
```bash
kubectl apply -f https://raw.githubusercontent.com/scylladb/scylla-operator/v1.11.0/deploy/operator.yaml
```

**Deploy single-node ScyllaDB:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: scylla
---
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-dev
  namespace: dev
spec:
  version: 5.4.0
  agentVersion: 3.2.0
  datacenter:
    name: dc1
    racks:
      - name: rack1
        members: 1
        storage:
          capacity: 30Gi
          storageClassName: nfs-db
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 2
            memory: 4Gi
EOF
```

**Wait for ScyllaDB:**
```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=scylla -n dev --timeout=10m
```

### 3.3 Deploy Gitea (Git Server)
**Duration:** 30 minutes

```bash
helm repo add gitea-charts https://dl.gitea.io/charts/

helm install gitea gitea-charts/gitea \
  --namespace dev \
  --set persistence.enabled=true \
  --set persistence.storageClass=nfs-client \
  --set persistence.size=50Gi \
  --set gitea.admin.username=admin \
  --set gitea.admin.password=adminpassword \
  --set gitea.admin.email=admin@yourdomain.com \
  --set postgresql.enabled=true \
  --set postgresql.global.storageClass=nfs-db
```

**Create Ingress:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea
  namespace: dev
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
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

### 3.4 Setup VS Code Remote Development
**Duration:** 20 minutes

**On your Windows machine:**

1. Install VS Code
2. Install Remote-SSH extension
3. Configure SSH config (`~/.ssh/config`):

```
Host pi-dev
    HostName 10.0.0.11
    User pi
    ForwardAgent yes
    
Host pi-control
    HostName 10.0.0.10
    User pi
```

4. Connect to `pi-dev` via Remote-SSH
5. Install Python, Node.js extensions in remote VS Code
6. Install development tools on Pi:

```bash
# On pi-worker-1
sudo apt install -y python3-pip python3-venv nodejs npm build-essential
```

**Configure kubectl access on Windows:**
```bash
# Copy kubeconfig from pi-control
scp pi@10.0.0.10:/etc/rancher/k3s/k3s.yaml ~/.kube/config

# Edit config: change server: https://127.0.0.1:6443 to server: https://10.0.0.10:6443
```

**✅ Phase 3 Complete:** Development environment ready

---