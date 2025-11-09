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

## Phase 6: Application Deployment (Ongoing)

### 6.1 Deploy Sample Python API
**Duration:** 30 minutes

**Create sample FastAPI app:**
```python
# app.py
from fastapi import FastAPI
import psycopg2
import redis

app = FastAPI()
r = redis.Redis(host='redis-master.prod.svc.cluster.local', password='redispassword')

@app.get("/")
async def root():
    return {"message": "Hello from Pi Cluster!"}

@app.get("/health")
async def health():
    return {"status": "healthy"}

@app.get("/counter")
async def counter():
    count = r.incr('hits')
    return {"hits": count}
```

**Dockerfile:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Kubernetes deployment:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-api
  namespace: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: python-api
  template:
    metadata:
      labels:
        app: python-api
    spec:
      containers:
      - name: api
        image: your-registry/python-api:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: python-api
  namespace: prod
spec:
  selector:
    app: python-api
  ports:
  - port: 80
    targetPort: 8000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: python-api
  namespace: prod
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  rules:
  - host: api.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: python-api
            port:
              number: 80
EOF
```

### 6.2 Deploy React Frontend
**Duration:** 30 minutes

Similar process:
1. Build React app
2. Create Dockerfile with nginx
3. Deploy to cluster
4. Configure ingress

---