# Ghost Blog Production Deployment Guide

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Ghost Namespace                       │
│  ┌──────────────┐      ┌────────────────────────────┐  │
│  │    Ghost     │─────▶│  PVC: ghost-content (10Gi) │  │
│  │  Deployment  │      └────────────────────────────┘  │
│  │              │                                        │
│  │ - CPU: 100m  │      ┌────────────────────────────┐  │
│  │ - RAM: 256Mi │─────▶│  Secret: ghost-secrets     │  │
│  │ - Image:     │      │  - db-password             │  │
│  │   ghost:     │      └────────────────────────────┘  │
│  │   5.130.5    │                                        │
│  └──────┬───────┘                                        │
│         │                                                │
│         │ Port 2368                                      │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │   Service    │                                        │
│  │ ghost:2368   │                                        │
│  └──────┬───────┘                                        │
└─────────┼─────────────────────────────────────────────────┘
          │
          │ Cross-namespace connection
          │
┌─────────▼─────────────────────────────────────────────────┐
│                     Prod Namespace                        │
│  ┌──────────────────────────────────────────────────┐    │
│  │  MySQL StatefulSet: mysql-ghost                  │    │
│  │                                                    │    │
│  │  - Image: mysql:8.0                               │    │
│  │  - CPU: 250m, RAM: 512Mi                          │    │
│  │  - Storage: 20Gi (nfs-client)                     │    │
│  │                                                    │    │
│  │  Service: mysql-ghost.prod.svc.cluster.local:3306│    │
│  └──────────────────────────────────────────────────┘    │
│                                                            │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Secret: mysql-ghost                              │    │
│  │  - root-password                                  │    │
│  │  - password (ghost user)                          │    │
│  └──────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────┘

External Access:
┌──────────────────────────────────────────────────────────┐
│  Ingress: blog.yourdomain.com                            │
│  - TLS: ghost-tls (cert-manager)                         │
│  - Middleware: Security headers                          │
└──────────────────────────────────────────────────────────┘
```

## Prerequisites

1. **Kubernetes cluster** with kubectl access
2. **StorageClass** `nfs-client` configured and available
3. **Traefik** ingress controller installed (or Cloudflare Tunnel)
4. **cert-manager** (optional, for automatic TLS certificates)
5. **Domain name** configured and pointing to your cluster

## File Overview

| File | Purpose | Namespace | Order |
|------|---------|-----------|-------|
| `00-ghost-namespace.yaml` | Creates Ghost namespace | ghost | 1 |
| `01-mysql-secret-template.yaml` | MySQL secret template (DO NOT APPLY) | prod | - |
| `02-ghost-secret-template.yaml` | Ghost secret template (DO NOT APPLY) | ghost | - |
| `03-mysql-statefulset.yaml` | MySQL database for Ghost | prod | 2 |
| `04-ghost-pvc.yaml` | Ghost content storage | ghost | 3 |
| `05-ghost-deployment.yaml` | Ghost application | ghost | 4 |
| `06-ghost-service.yaml` | Ghost service | ghost | 5 |
| `07-ghost-ingress.yaml` | Ghost ingress with TLS | ghost | 6 |

## Step-by-Step Deployment

### Step 1: Create Namespaces

```bash
# Create Ghost namespace
kubectl apply -f 00-ghost-namespace.yaml

# Verify prod namespace exists (should already exist in your cluster)
kubectl get namespace prod
```

### Step 2: Create Secrets

**IMPORTANT:** Never commit actual secrets to Git!

#### 2a. Create MySQL Secret (in prod namespace)

```bash
# Generate strong passwords
MYSQL_ROOT_PASS=$(openssl rand -base64 32)
MYSQL_GHOST_PASS=$(openssl rand -base64 32)

# Create the secret
kubectl create secret generic mysql-ghost \
  --from-literal=root-password="${MYSQL_ROOT_PASS}" \
  --from-literal=password="${MYSQL_GHOST_PASS}" \
  -n prod

# CRITICAL: Save these passwords in your password manager!
echo "MySQL Root Password: ${MYSQL_ROOT_PASS}"
echo "MySQL Ghost Password: ${MYSQL_GHOST_PASS}"
```

#### 2b. Create Ghost Secret (in ghost namespace)

```bash
# Use the SAME password as the Ghost user password from MySQL
# This allows Ghost to connect to MySQL
kubectl create secret generic ghost-secrets \
  --from-literal=db-password="${MYSQL_GHOST_PASS}" \
  -n ghost

# Optional: Add mail configuration later
# kubectl create secret generic ghost-secrets \
#   --from-literal=db-password="${MYSQL_GHOST_PASS}" \
#   --from-literal=mail-user="your-email@gmail.com" \
#   --from-literal=mail-password="your-app-password" \
#   -n ghost --dry-run=client -o yaml | kubectl apply -f -
```

#### 2c. Verify Secrets

```bash
# Check secrets exist
kubectl get secret mysql-ghost -n prod
kubectl get secret ghost-secrets -n ghost

# Verify they have the correct keys (don't display values)
kubectl get secret mysql-ghost -n prod -o jsonpath='{.data}' | jq 'keys'
kubectl get secret ghost-secrets -n ghost -o jsonpath='{.data}' | jq 'keys'
```

### Step 3: Deploy MySQL Database

```bash
# Deploy MySQL StatefulSet
kubectl apply -f 03-mysql-statefulset.yaml

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app=mysql-ghost -n prod --timeout=300s

# Run the init Job
kubectl apply -f 03b-mysql-init-job.yaml

# Watch it work
kubectl logs -n prod -l app=mysql-init -f

# Verify success
kubectl get job mysql-init -n prod  # Should show "1/1" completions

# Check MySQL status
kubectl get statefulset mysql-ghost -n prod
kubectl get pod -l app=mysql-ghost -n prod
kubectl get pvc -n prod | grep mysql

# Check MySQL logs (should show successful startup)
kubectl logs -l app=mysql-ghost -n prod --tail=50

# Test MySQL connectivity (optional)
kubectl run mysql-client --image=mysql:8.0 -it --rm --restart=Never -n prod -- \
  mysql -h mysql-ghost.prod.svc.cluster.local -u ghost -p${MYSQL_GHOST_PASS} -e "SHOW DATABASES;"
```

### Step 4: Update Ghost Configuration

**BEFORE deploying Ghost, update the domain in these files:**

Edit `05-ghost-deployment.yaml`:
```yaml
# Line ~60 - Update this:
- name: url
  value: "https://blog.yourdomain.com"  # CHANGE THIS!
```

Edit `07-ghost-ingress.yaml`:
```yaml
# Lines for host - Update these:
- host: blog.yourdomain.com  # CHANGE THIS!
```

### Step 5: Deploy Ghost Application

```bash
# Create Ghost PVC
kubectl apply -f 04-ghost-pvc.yaml

# Verify PVC is bound
kubectl get pvc ghost-content -n ghost

# Deploy Ghost
kubectl apply -f 05-ghost-deployment.yaml

# Deploy Ghost Service
kubectl apply -f 06-ghost-service.yaml

# Wait for Ghost to be ready
kubectl wait --for=condition=available deployment/ghost -n ghost --timeout=300s

# Check Ghost status
kubectl get deployment ghost -n ghost
kubectl get pod -l app=ghost -n ghost
kubectl logs -l app=ghost -n ghost --tail=50
```

### Step 6: Configure External Access

#### Option A: Using Traefik Ingress (recommended for local/VPN access)

```bash
# Deploy Ingress
kubectl apply -f 07-ghost-ingress.yaml

# Check Ingress status
kubectl get ingress ghost -n ghost
kubectl describe ingress ghost -n ghost

# If using cert-manager, check certificate
kubectl get certificate -n ghost
```

#### Option B: Using Cloudflare Tunnel (recommended for public access)

If you prefer Cloudflare Tunnel (more secure), skip the Ingress and configure:

```bash
# Add to your Cloudflare Tunnel configuration:
# Subdomain: blog
# Service: http://ghost.ghost.svc.cluster.local:2368
```

### Step 7: Initial Setup

1. **Access Ghost admin**: `https://blog.yourdomain.com/ghost`
2. **Create your admin account** (first time only)
3. **Configure your blog**:
   - Site title
   - Site description
   - Publication icon/logo
   - Theme selection

### Step 8: Backup Strategy

```bash
# Create backup directory
mkdir -p ~/ghost-backups

# Backup MySQL database
kubectl exec -n prod mysql-ghost-0 -- \
  mysqldump -u root -p${MYSQL_ROOT_PASS} ghost | \
  gzip > ~/ghost-backups/ghost-db-$(date +%Y%m%d-%H%M%S).sql.gz

# Backup Ghost content (from PVC)
kubectl cp ghost/<ghost-pod-name>:/var/lib/ghost/content \
  ~/ghost-backups/ghost-content-$(date +%Y%m%d-%H%M%S)

# Backup secrets (encrypted)
kubectl get secret mysql-ghost -n prod -o yaml > ~/ghost-backups/mysql-secret.yaml
kubectl get secret ghost-secrets -n ghost -o yaml > ~/ghost-backups/ghost-secret.yaml
# Encrypt these files
gpg -c ~/ghost-backups/mysql-secret.yaml
gpg -c ~/ghost-backups/ghost-secret.yaml
# Delete unencrypted versions
rm ~/ghost-backups/*-secret.yaml
```

## Verification Checklist

- [ ] MySQL StatefulSet is running in `prod` namespace
- [ ] MySQL PVC is bound and has 20Gi storage
- [ ] Ghost Deployment is running in `ghost` namespace
- [ ] Ghost PVC is bound and has 10Gi storage
- [ ] Ghost pod logs show successful database connection
- [ ] Ghost service is accessible within cluster
- [ ] Ingress/Cloudflare Tunnel is configured
- [ ] Can access `https://blog.yourdomain.com/ghost`
- [ ] Can create admin account
- [ ] Can view public homepage
- [ ] Secrets are backed up and encrypted

## Troubleshooting

### Ghost can't connect to MySQL

```bash
# Check if MySQL is running
kubectl get pod -l app=mysql-ghost -n prod

# Test DNS resolution from Ghost pod
GHOST_POD=$(kubectl get pod -l app=ghost -n ghost -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n ghost $GHOST_POD -- nslookup mysql-ghost.prod.svc.cluster.local

# Test MySQL connection from Ghost pod
kubectl exec -n ghost $GHOST_POD -- nc -zv mysql-ghost.prod.svc.cluster.local 3306

# Check Ghost logs
kubectl logs -l app=ghost -n ghost --tail=100

# Verify secrets match
kubectl get secret mysql-ghost -n prod -o jsonpath='{.data.password}' | base64 -d
kubectl get secret ghost-secrets -n ghost -o jsonpath='{.data.db-password}' | base64 -d
# These should be IDENTICAL
```

### Ghost pod is crash looping

```bash
# Check pod events
kubectl describe pod -l app=ghost -n ghost

# Check logs with previous instance
kubectl logs -l app=ghost -n ghost --previous

# Check PVC is mounted correctly
kubectl describe pod -l app=ghost -n ghost | grep -A 10 Volumes
```

### Can't access Ghost externally

```bash
# Check Ingress
kubectl get ingress ghost -n ghost
kubectl describe ingress ghost -n ghost

# Check if Traefik is routing correctly
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik

# Test internal access
kubectl port-forward -n ghost svc/ghost 8080:2368
# Then access http://localhost:8080 in browser
```

### Database performance issues

```bash
# Check MySQL resource usage
kubectl top pod -l app=mysql-ghost -n prod

# Check slow query log
kubectl exec -n prod mysql-ghost-0 -- cat /var/lib/mysql/slow-query.log

# Increase resources if needed (edit 03-mysql-statefulset.yaml)
```

## Maintenance Operations

### Update Ghost Version

```bash
# Edit 05-ghost-deployment.yaml and update image version
# Then apply changes
kubectl apply -f 05-ghost-deployment.yaml

# Watch the rollout
kubectl rollout status deployment/ghost -n ghost

# Rollback if needed
kubectl rollout undo deployment/ghost -n ghost
```

### Scale Ghost (for high traffic)

```bash
# Edit replicas in 05-ghost-deployment.yaml
# Or scale directly:
kubectl scale deployment ghost -n ghost --replicas=2

# Verify
kubectl get deployment ghost -n ghost
```

### Rotate Secrets

```bash
# Generate new password
NEW_PASS=$(openssl rand -base64 32)

# Update MySQL
kubectl exec -n prod mysql-ghost-0 -- \
  mysql -u root -p${MYSQL_ROOT_PASS} \
  -e "ALTER USER 'ghost'@'%' IDENTIFIED BY '${NEW_PASS}';"

# Update Ghost secret
kubectl create secret generic ghost-secrets \
  --from-literal=db-password="${NEW_PASS}" \
  -n ghost --dry-run=client -o yaml | kubectl apply -f -

# Restart Ghost to pick up new secret
kubectl rollout restart deployment/ghost -n ghost

# Save new password in password manager!
```

## Resource Requirements

### Minimum:
- **MySQL**: 512Mi RAM, 250m CPU
- **Ghost**: 256Mi RAM, 100m CPU
- **Storage**: 30Gi total (20Gi MySQL + 10Gi Ghost content)

### Recommended for Production:
- **MySQL**: 1Gi RAM, 500m CPU
- **Ghost**: 512Mi RAM (2 replicas)
- **Storage**: 50Gi total with room to grow

## Security Considerations

1. **Secrets Management**:
   - Never commit secrets to Git
   - Use encrypted backups
   - Store passwords in password manager
   - Rotate every 6 months

2. **Network Policies** (optional but recommended):
   ```bash
   # Create network policy to restrict Ghost access to MySQL only
   kubectl apply -f - <<EOF
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: mysql-ghost-access
     namespace: prod
   spec:
     podSelector:
       matchLabels:
         app: mysql-ghost
     ingress:
     - from:
       - namespaceSelector:
           matchLabels:
             name: ghost
       ports:
       - protocol: TCP
         port: 3306
   EOF
   ```

3. **Regular Updates**:
   - Keep Ghost updated
   - Keep MySQL updated
   - Monitor security advisories

4. **Backup Strategy**:
   - Daily automated backups
   - Test restore procedures
   - Store backups off-cluster

## Complete Deployment Command

For a fresh deployment (after creating secrets):

```bash
# One-liner to deploy everything in order
kubectl apply -f 00-ghost-namespace.yaml && \
kubectl apply -f 03-mysql-statefulset.yaml && \
kubectl wait --for=condition=ready pod -l app=mysql-ghost -n prod --timeout=300s && \
kubectl apply -f 04-ghost-pvc.yaml && \
kubectl apply -f 05-ghost-deployment.yaml && \
kubectl apply -f 06-ghost-service.yaml && \
kubectl apply -f 07-ghost-ingress.yaml && \
echo "✅ Ghost blog deployed successfully!"
```

## Support Resources

- **Ghost Documentation**: https://ghost.org/docs/
- **MySQL Documentation**: https://dev.mysql.com/doc/
- **Kubernetes Documentation**: https://kubernetes.io/docs/
- **Your Project Docs**: Check your project knowledge base

---

**Last Updated**: November 2025
**Ghost Version**: 5.130.5
**MySQL Version**: 8.0
