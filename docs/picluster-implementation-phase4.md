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

## Phase 4: Production Environment (Day 2-3)

### 4.1 Deploy Production PostgreSQL (HA)
**Duration:** 45 minutes

```bash
helm install postgresql-prod bitnami/postgresql \
  --namespace prod \
  --set auth.postgresPassword=prodpassword \
  --set auth.database=proddb \
  --set architecture=replication \
  --set primary.persistence.storageClass=nfs-db \
  --set primary.persistence.size=50Gi \
  --set readReplicas.replicaCount=1 \
  --set readReplicas.persistence.storageClass=nfs-db \
  --set readReplicas.persistence.size=50Gi \
  --set primary.resources.requests.memory=1Gi \
  --set primary.resources.requests.cpu=500m \
  --set metrics.enabled=true
```

### 4.2 Deploy Production ScyllaDB (3-node cluster)
**Duration:** 1 hour

```bash
cat <<EOF | kubectl apply -f -
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-prod
  namespace: prod
spec:
  version: 5.4.0
  agentVersion: 3.2.0
  datacenter:
    name: dc1
    racks:
      - name: rack1
        members: 3
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

### 4.3 Deploy Ghost Blog
**Duration:** 45 minutes

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ghost-content
  namespace: prod
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
  namespace: prod
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
        - name: database__client
          value: "sqlite3"
        - name: database__connection__filename
          value: "/var/lib/ghost/content/data/ghost.db"
        - name: NODE_ENV
          value: "production"
        volumeMounts:
        - name: ghost-content
          mountPath: /var/lib/ghost/content
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
      volumes:
      - name: ghost-content
        persistentVolumeClaim:
          claimName: ghost-content
---
apiVersion: v1
kind: Service
metadata:
  name: ghost
  namespace: prod
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
  namespace: prod
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
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

**Access Ghost:**
- Navigate to `https://blog.yourdomain.com/ghost`
- Complete setup wizard

### 4.4 Deploy Redis Cache
**Duration:** 15 minutes

```bash
helm install redis bitnami/redis \
  --namespace prod \
  --set auth.password=redispassword \
  --set master.persistence.enabled=true \
  --set master.persistence.storageClass=nfs-db \
  --set master.persistence.size=10Gi \
  --set replica.replicaCount=0 \
  --set master.resources.requests.memory=256Mi \
  --set master.resources.requests.cpu=100m
```

### 4.5 Setup GitHub Actions CI/CD
**Duration:** 30 minutes

**Create deployment script on cluster:**
```bash
cat <<EOF > /home/pi/deploy-app.sh
#!/bin/bash
NAMESPACE=\$1
APP_NAME=\$2
IMAGE=\$3

kubectl set image deployment/\$APP_NAME \$APP_NAME=\$IMAGE -n \$NAMESPACE
kubectl rollout status deployment/\$APP_NAME -n \$NAMESPACE
EOF

chmod +x /home/pi/deploy-app.sh
```

**GitHub Actions workflow example (`.github/workflows/deploy.yml`):**
```yaml
name: Deploy to Production

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker image
        run: |
          docker build -t your-app:${{ github.sha }} .
          
      - name: Push to registry
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker push your-app:${{ github.sha }}
          
      - name: Deploy to cluster
        uses: appleboy/ssh-action@master
        with:
          host: 10.0.0.10
          username: pi
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            /home/pi/deploy-app.sh prod your-app your-app:${{ github.sha }}
```

**✅ Phase 4 Complete:** Production environment operational

---