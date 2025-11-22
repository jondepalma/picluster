# Claude Code Development Context

**Project Environment:** Raspberry Pi K3s Cluster Development Setup  
**Last Updated:** 2025-11-21

---

## Infrastructure Overview

### Cluster Nodes
- **pi-control** (10.0.0.10) - K3s control plane
  - 8GB RAM, 1TB NVMe SSD at `/mnt/nvme`
  - Runs critical infrastructure pods
  - Storage: NVMe for high-performance workloads
  
- **pi-worker-1** (10.0.0.11) - **DEV ENVIRONMENT NODE** ⭐
  - 8GB RAM, 256GB SD card
  - **Bare-metal development** happens here
  - kubectl configured to access cluster
  - Development tools: Python 3.11+, Node.js 20+, Docker
  
- **pi-worker-2** (10.0.0.12) - Worker node
  - 8GB RAM, 256GB SD card
  - Runs application workloads

### Network Configuration
- **Internal Network:** 10.0.0.0/24
- **QNAP NAS:** 10.0.0.5 (NFS storage)
- **MetalLB IP Pool:** 10.0.0.30-10.0.0.50
- **VPN:** Tailscale (for remote access)
- **Public Access:** Cloudflare Tunnel

---

## Kubernetes Configuration

### Namespaces
- `dev` - Development environment (databases, test apps)
- `prod` - Production environment
- `monitoring` - Prometheus, Grafana
- `traefik` - Ingress controller

### Storage Classes
- `local-path` (default) - SD card storage, temporary data
- `local-path-nvme` - High-performance NVMe on pi-control
- `nfs-client` - Persistent bulk storage on NAS
- `nfs-db` - Database-optimized NFS storage

### Ingress Controller
- **Traefik** in `traefik` namespace
- LoadBalancer IP: Check with `kubectl get svc -n traefik traefik`
- Public access via Cloudflare Tunnel

---

## Database Services in Cluster

### Development PostgreSQL (Namespace: `dev`)
**Service Name:** `postgresql-dev.dev.svc.cluster.local`
- **Host (from inside cluster):** `postgresql-dev.dev.svc.cluster.local`
- **Host (from bare-metal via port-forward):** `localhost` or `10.0.0.11`
- **Port:** 5432
- **Database:** `devdb`
- **Username:** `postgres`
- **Password:** Get with: `kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 -d`
- **Connection String (in-cluster):**
  ```
  postgresql://postgres:<password>@postgresql-dev.dev.svc.cluster.local:5432/devdb
  ```

### Development ScyllaDB (Namespace: `dev`)
**Service Name:** `scylla-dev-client.dev.svc.cluster.local`
- **Host (from inside cluster):** `scylla-dev-client.dev.svc.cluster.local`
- **Host (from bare-metal via port-forward):** `localhost` or `10.0.0.11`
- **Port:** 9042 (CQL)
- **Username:** `cassandra`
- **Password:** Get with: `kubectl get secret scylladb-dev -n dev -o jsonpath='{.data.cassandra-password}' | base64 -d`
- **Keyspace:** Create as needed

---

## Development Workflow

### Where Code Lives
- **Development Directory:** `/home/pi/dev-projects/` on pi-worker-1 (10.0.0.11)
- **VS Code Remote-SSH:** Connect to `pi-dev` host
- **Git Repositories:** Clone to dev-projects directory

### Accessing Cluster Databases from Bare-Metal

#### Option 1: Port-Forwarding (Quick Testing)
```bash
# On pi-worker-1
kubectl port-forward -n dev svc/postgresql-dev 5432:5432 &
kubectl port-forward -n dev svc/scylla-dev-client 9042:9042 &

# Now connect to localhost:5432 and localhost:9042
```

#### Option 2: Deploy to Cluster (Production-Like)
```bash
# Build Docker image
docker build -t your-username/app-name:version .

# Push to registry
docker push your-username/app-name:version

# Deploy to dev namespace
kubectl apply -f deployment.yaml
```

### Environment Variables Pattern

**For Bare-Metal Development (with port-forwarding):**
```bash
# .env file
DATABASE_URL=postgresql://postgres:devpassword@localhost:5432/devdb
SCYLLA_HOSTS=localhost
SCYLLA_PORT=9042
SCYLLA_KEYSPACE=dev_keyspace
ENVIRONMENT=development
```

**For Containerized Deployment in Cluster:**
```yaml
# ConfigMap or deployment.yaml
env:
  - name: DATABASE_URL
    value: "postgresql://postgres:password@postgresql-dev.dev.svc.cluster.local:5432/devdb"
  - name: SCYLLA_HOSTS
    value: "scylla-dev-client.dev.svc.cluster.local"
  - name: SCYLLA_PORT
    value: "9042"
  - name: ENVIRONMENT
    value: "production"
```

---

## Common Commands

### kubectl Access
```bash
# Configured on pi-worker-1 at ~/.kube/config
# Can run kubectl commands directly

# Get pods in dev namespace
kubectl get pods -n dev

# View logs
kubectl logs <pod-name> -n dev

# Port-forward service
kubectl port-forward -n dev svc/<service-name> <local-port>:<remote-port>

# Execute into pod
kubectl exec -it <pod-name> -n dev -- /bin/bash
```

### Docker Workflow
```bash
# Build multi-arch image (for ARM64 Raspberry Pi)
docker buildx build --platform linux/arm64 -t your-username/app:latest .

# Or build locally on Pi
docker build -t your-username/app:latest .

# Push to Docker Hub
docker push your-username/app:latest
```

### Database Access
```bash
# Get PostgreSQL password
kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 -d

# Get ScyllaDB password
kubectl get secret scylladb-dev -n dev -o jsonpath='{.data.cassandra-password}' | base64 -d

# Connect to PostgreSQL (from inside cluster)
kubectl exec -it postgresql-dev-0 -n dev -- psql -U postgres -d devdb

# Connect to ScyllaDB (from inside cluster)
PASSWORD=$(kubectl get secret scylladb-dev -n dev -o jsonpath='{.data.cassandra-password}' | base64 -d)
kubectl exec -it scylla-dev-dc1-rack1-0 -n dev -c scylla -- cqlsh -u cassandra -p $PASSWORD
```

---

## Application Deployment Pattern

### Local Development on pi-worker-1
1. Develop code in `/home/pi/dev-projects/your-app/`
2. Port-forward databases to localhost
3. Run app locally: `python app.py` or `npm start`
4. Test against local ports

### Deploy to Cluster
1. Create Dockerfile
2. Build image: `docker build -t your-username/app:version .`
3. Push to registry: `docker push your-username/app:version`
4. Create Kubernetes manifests (deployment.yaml, service.yaml)
5. Deploy: `kubectl apply -f deployment.yaml`
6. Expose via Ingress if needed for external access

---

## Secrets Management

### Getting Secrets
```bash
# View all secrets in namespace
kubectl get secrets -n dev

# Get specific secret value
kubectl get secret <secret-name> -n dev -o jsonpath='{.data.<key>}' | base64 -d

# Create secret from literal
kubectl create secret generic my-secret \
  --from-literal=api-key=myapikey123 \
  -n dev
```

### Using Secrets in Applications
```yaml
env:
  - name: DATABASE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgresql-dev
        key: postgres-password
```

---

## Monitoring & Debugging

### Check Resource Usage
```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods -n dev
```

### View Logs
```bash
# Single pod
kubectl logs <pod-name> -n dev

# Follow logs
kubectl logs -f <pod-name> -n dev

# Previous container logs (if pod crashed)
kubectl logs <pod-name> -n dev --previous
```

### Debugging Pods
```bash
# Describe pod (see events)
kubectl describe pod <pod-name> -n dev

# Execute into running pod
kubectl exec -it <pod-name> -n dev -- /bin/bash

# Run debug pod
kubectl run debug --rm -it --image=busybox -n dev -- sh
```

---

## Best Practices for This Environment

### Code Structure
- Keep environment-specific configs in separate files
- Use environment variables for configuration
- Create `.env.example` files for documentation
- Never commit secrets to Git

### Docker Images
- **Always build for ARM64:** `--platform linux/arm64`
- Keep images small (use Alpine base images when possible)
- Tag images with version numbers, not just `latest`

### Database Connections
- Use connection pooling
- Implement retry logic for cluster connectivity
- Close connections properly
- Use separate credentials for dev/prod

### Resource Limits
- Always set resource requests/limits in deployments
- Monitor memory usage (Pis have 8GB RAM)
- Use persistent volumes for data that needs to survive pod restarts

---

## Troubleshooting

### "Connection refused" errors
- Check if port-forwarding is active: `ps aux | grep kubectl`
- Verify service exists: `kubectl get svc -n dev`
- Check pod status: `kubectl get pods -n dev`

### kubectl not working on pi-worker-1
```bash
# Re-copy kubeconfig
scp pi@10.0.0.10:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1/10.0.0.10/g' ~/.kube/config
chmod 600 ~/.kube/config
```

### Pod won't start
```bash
# Check events
kubectl describe pod <pod-name> -n dev

# Common issues:
# - ImagePullBackOff: Image not found or wrong architecture
# - CrashLoopBackOff: Application error, check logs
# - Pending: Resource constraints or storage issues
```

### Database connection issues
```bash
# Test from debug pod
kubectl run debug --rm -it --image=busybox -n dev -- sh
# Then: nc -zv postgresql-dev.dev.svc.cluster.local 5432
```

---

## External Resources

- **Project Documentation:** `/mnt/project/` directory
- **K3s Docs:** https://docs.k3s.io
- **Kubectl Cheat Sheet:** https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **Traefik Docs:** https://doc.traefik.io/traefik/
- **ScyllaDB Docs:** https://docs.scylladb.com

---

## Quick Reference Card

```bash
# Port-forward all dev databases
kubectl port-forward -n dev svc/postgresql-dev 5432:5432 &
kubectl port-forward -n dev svc/scylla-dev-client 9042:9042 &

# Get database passwords
kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 -d
kubectl get secret scylladb-dev -n dev -o jsonpath='{.data.cassandra-password}' | base64 -d

# Deploy app to cluster
docker build -t myapp:v1 .
docker push myapp:v1
kubectl apply -f deployment.yaml
kubectl get pods -n dev
kubectl logs -f <pod-name> -n dev

# Common checks
kubectl get nodes
kubectl get pods -n dev
kubectl get svc -n dev
kubectl top nodes
```
