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

## Phase 5: Monitoring & Backups (Day 3)

### 5.1 Deploy Prometheus & Grafana
**Duration:** 45 minutes

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=nfs-client \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=nfs-client \
  --set grafana.persistence.size=5Gi \
  --set grafana.adminPassword=grafanapassword
```

**Create Ingress for Grafana:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  rules:
  - host: grafana.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kube-prometheus-stack-grafana
            port:
              number: 80
EOF
```

### 5.2 Setup Automated Backups
**Duration:** 30 minutes

**Create backup CronJob:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: prod
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15-alpine
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h postgresql-prod-primary -U postgres -d proddb | gzip > /backups/postgres-\$(date +%Y%m%d).sql.gz
              find /backups -name "postgres-*.sql.gz" -mtime +7 -delete
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgresql-prod
                  key: postgres-password
            volumeMounts:
            - name: backups
              mountPath: /backups
          volumes:
          - name: backups
            nfs:
              server: 10.0.0.5
              path: /share/kubernetes-backups
          restartPolicy: OnFailure
EOF
```

### 5.3 Setup Health Monitoring
**Duration:** 20 minutes

**Create simple uptime monitor:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: uptime-monitor
  namespace: monitoring
data:
  monitor.sh: |
    #!/bin/bash
    while true; do
      curl -sf https://blog.yourdomain.com > /dev/null || echo "Ghost down!"
      curl -sf https://api.yourdomain.com/health > /dev/null || echo "API down!"
      sleep 60
    done
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: uptime-monitor
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: uptime-monitor
  template:
    metadata:
      labels:
        app: uptime-monitor
    spec:
      containers:
      - name: monitor
        image: curlimages/curl:latest
        command: ["/bin/sh", "/scripts/monitor.sh"]
        volumeMounts:
        - name: scripts
          mountPath: /scripts
      volumes:
      - name: scripts
        configMap:
          name: uptime-monitor
          defaultMode: 0755
EOF
```

**✅ Phase 5 Complete:** Monitoring and backups configured

---