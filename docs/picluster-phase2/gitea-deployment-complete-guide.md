# Gitea Deployment Complete Guide

**Complete guide for deploying Gitea to Kubernetes with external PostgreSQL, proper secrets management, and NFS storage fixes.**

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [PostgreSQL Setup](#postgresql-setup)
4. [Secrets Creation](#secrets-creation)
5. [NFS Permissions Fix](#nfs-permissions-fix)
6. [Inline Config Secret](#inline-config-secret)
7. [Helm Installation](#helm-installation)
8. [Verification](#verification)
9. [Troubleshooting](#troubleshooting)
10. [Maintenance](#maintenance)

---

## Overview

This guide documents the complete, working deployment of Gitea using:
- **External PostgreSQL** (existing PostgreSQL in dev namespace)
- **NFS storage** for Gitea data
- **Kubernetes secrets** for credentials (not hardcoded)
- **Cloudflare Tunnel** for external access
- **Manual NFS permission fixes** (workaround for provisioner limitations)

---

## Prerequisites

- [ ] Kubernetes cluster running (K3s)
- [ ] PostgreSQL deployed in `dev` namespace
- [ ] NFS storage provisioner configured
- [ ] Helm installed
- [ ] kubectl access

### Add Gitea Helm Repository

```bash
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
```

---

## PostgreSQL Setup

### Step 1: Get PostgreSQL Admin Password

```bash
# Get the postgres superuser password
POSTGRES_PASS=$(kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 --decode)

echo "PostgreSQL admin password: $POSTGRES_PASS"
```

### Step 2: Generate Gitea Database Password

```bash
# Generate a strong password for Gitea database user
GITEA_DB_PASSWORD=$(openssl rand -base64 32)

echo "Gitea DB Password (SAVE THIS): $GITEA_DB_PASSWORD"
```

### Step 3: Create Gitea Database and User

```bash
# Connect to PostgreSQL and create database
kubectl exec -it -n dev postgresql-dev-0 -- bash -c "PGPASSWORD='$POSTGRES_PASS' psql -U postgres" <<EOF
CREATE DATABASE gitea;
CREATE USER gitea WITH PASSWORD '$GITEA_DB_PASSWORD';
GRANT ALL PRIVILEGES ON DATABASE gitea TO gitea;
ALTER DATABASE gitea OWNER TO gitea;
\q
EOF

# Verify creation
kubectl exec -it -n dev postgresql-dev-0 -- bash -c "PGPASSWORD='$POSTGRES_PASS' psql -U postgres -c '\l'" | grep gitea
kubectl exec -it -n dev postgresql-dev-0 -- bash -c "PGPASSWORD='$POSTGRES_PASS' psql -U postgres -c '\du'" | grep gitea
```

### Step 4: Test Database Connection

```bash
# Verify you can connect as gitea user
kubectl run test-db --rm -it --image=postgres:15 -n dev --restart=Never -- \
  bash -c "PGPASSWORD='$GITEA_DB_PASSWORD' psql -h postgresql-dev.dev.svc.cluster.local -U gitea -d gitea -c 'SELECT version();'"

# Should output PostgreSQL version - connection successful!
```

---

## Secrets Creation

### Step 1: Create Gitea Admin Secret

```bash
# Generate admin password
ADMIN_PASSWORD=$(openssl rand -base64 24)

echo "Gitea Admin Password (SAVE THIS): $ADMIN_PASSWORD"

# Create secret
kubectl create secret generic gitea-admin \
  --from-literal=username=admin \
  --from-literal=password="$ADMIN_PASSWORD" \
  --from-literal=email=admin@jondepalma.net \
  -n dev

# Verify
kubectl get secret gitea-admin -n dev
```

### Step 2: Create Database Password Secret

```bash
# Create secret with database password
kubectl create secret generic gitea-external-db \
  --from-literal=password="$GITEA_DB_PASSWORD" \
  -n dev

# Verify
kubectl get secret gitea-external-db -n dev
```

### Step 3: Verify All Secrets

```bash
# List secrets
kubectl get secrets -n dev | grep gitea

# Should show:
# gitea-admin         Opaque   3      Xs
# gitea-external-db   Opaque   1      Xs
```

---

## NFS Permissions Fix

### The Problem

The NFS provisioner creates directories owned by `root:root`, but Gitea's rootless container runs as UID/GID `1000`. This causes "Permission denied" errors.

### The Solution: Manual Permission Fix

**Note:** The Helm chart's `initPreScript` doesn't work reliably, so we fix permissions manually after PVC creation.

### Step 1: Wait for PVC Creation

```bash
# After installing Gitea, wait for PVC to be created
kubectl get pvc -n dev | grep gitea-shared-storage

# Note the PVC name/ID in the VOLUME column
```

### Step 2: Fix Permissions on NAS

```bash
# Mount NFS share
sudo mkdir -p /mnt/nfs-fix
sudo mount -t nfs 10.0.0.5:/share/kubernetes-pv /mnt/nfs-fix

# List Gitea directories
ls -la /mnt/nfs-fix/ | grep gitea-shared

# You'll see something like:
# drwxrwxrwx+ 2 root root 4096 Nov 18 21:38 dev-gitea-shared-storage-pvc-XXXXXXXXX

# Fix permissions (replace with your actual directory name)
sudo chown -R 1000:1000 /mnt/nfs-fix/dev-gitea-shared-storage-pvc-*
sudo chmod -R 755 /mnt/nfs-fix/dev-gitea-shared-storage-pvc-*

# Verify the fix
ls -la /mnt/nfs-fix/dev-gitea-shared-storage-pvc-*
# Should now show: drwxr-xr-x ... 1000 1000 ...

# Unmount
sudo umount /mnt/nfs-fix
sudo rmdir /mnt/nfs-fix
```

### Step 3: Restart Gitea Pod

```bash
# Delete pod to trigger restart with correct permissions
kubectl delete pod -n dev -l app.kubernetes.io/name=gitea

# Watch it restart
kubectl get pods -n dev -w
```

### Automation Script (Optional)

Create a helper script for future permission fixes:

```bash
cat > ~/fix-gitea-nfs-permissions.sh <<'EOF'
#!/bin/bash
echo "🔧 Fixing Gitea NFS Permissions"

# Mount NFS
sudo mkdir -p /mnt/nfs-fix
sudo mount -t nfs 10.0.0.5:/share/kubernetes-pv /mnt/nfs-fix

# Fix permissions
echo "Fixing permissions for UID/GID 1000..."
sudo chown -R 1000:1000 /mnt/nfs-fix/dev-gitea-shared-storage-pvc-*
sudo chmod -R 755 /mnt/nfs-fix/dev-gitea-shared-storage-pvc-*

# Verify
echo "Verifying permissions:"
ls -la /mnt/nfs-fix/dev-gitea-shared-storage-pvc-*

# Unmount
sudo umount /mnt/nfs-fix
sudo rmdir /mnt/nfs-fix

echo "✅ Permissions fixed!"
echo "Now restart the pod: kubectl delete pod -n dev -l app.kubernetes.io/name=gitea"
EOF

chmod +x ~/fix-gitea-nfs-permissions.sh
```

---

## Inline Config Secret

### The Problem

The Gitea Helm chart has issues with database configuration:
1. Concatenates `DB_TYPE` + `SCHEMA` into `postgresschema`
2. Environment variables don't properly override config
3. Password from secret not injected correctly

### The Solution: Inline Config Secret

We create a separate secret with the full database configuration (including password) and inject it via `additionalConfigSources`.

### Create Inline Config Secret

```bash
# Get the database password (if you don't have it in a variable)
GITEA_DB_PASSWORD=$(kubectl get secret gitea-external-db -n dev -o jsonpath='{.data.password}' | base64 --decode)

# Create inline config secret with Helm labels
cat > /tmp/gitea-inline-config.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitea-inline-config
  namespace: dev
  labels:
    app.kubernetes.io/managed-by: Helm
  annotations:
    meta.helm.sh/release-name: gitea
    meta.helm.sh/release-namespace: dev
type: Opaque
stringData:
  database: |
    DB_TYPE=postgres
    HOST=postgresql-dev.dev.svc.cluster.local:5432
    NAME=gitea
    USER=gitea
    PASSWD=${GITEA_DB_PASSWORD}
    SSL_MODE=disable
EOF

# Apply the secret
kubectl apply -f /tmp/gitea-inline-config.yaml

# Verify
kubectl get secret gitea-inline-config -n dev

# Clean up temp file
rm /tmp/gitea-inline-config.yaml
```

**Important Notes:**
- The secret has **Helm labels and annotations** - this allows Helm to manage it
- The password is **NOT in Git** - only in Kubernetes
- Format is INI-style config that Gitea reads directly
- `SSL_MODE=disable` is correct for internal cluster communication

---

## Helm Installation

### Step 1: Create Values File

```bash
# Create directory for Gitea manifests
mkdir -p ~/pi-cluster-infrastructure/applications/gitea

# Create values file
cat > ~/pi-cluster-infrastructure/applications/gitea/values.yaml <<'EOF'
# Gitea Helm Chart Values
# Using external PostgreSQL and NFS storage

persistence:
  enabled: true
  storageClass: nfs-client
  size: 50Gi

gitea:
  admin:
    existingSecret: gitea-admin
  
  # Use inline config for database (includes password from secret)
  additionalConfigSources:
    - secret:
        secretName: gitea-inline-config
  
  config:
    server:
      DOMAIN: git.jondepalma.net
      ROOT_URL: https://git.jondepalma.net
      SSH_DOMAIN: git.jondepalma.net
  
  # Security context for rootless Gitea
  podSecurityContext:
    fsGroup: 1000
  
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000

# Disable built-in databases
postgresql:
  enabled: false

postgresql-ha:
  enabled: false

redis-cluster:
  enabled: false

# Ingress configuration
ingress:
  enabled: true
  className: traefik
  hosts:
    - host: git.jondepalma.net
      paths:
        - path: /
          pathType: Prefix

# Resource limits
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
EOF
```

### Step 2: Install Gitea

```bash
# Install Gitea with Helm
helm install gitea gitea-charts/gitea \
  --namespace dev \
  -f ~/pi-cluster-infrastructure/applications/gitea/values.yaml

# Watch pods start
kubectl get pods -n dev -w

# Expected sequence:
# 1. init-directories: Creates directory structure
# 2. init-app-ini: Generates app.ini configuration
# 3. configure-gitea: Runs database migrations
# 4. gitea: Main application starts
```

### Step 3: Fix NFS Permissions (If Needed)

```bash
# If you see CrashLoopBackOff, fix permissions:
~/fix-gitea-nfs-permissions.sh

# Or manually:
sudo mkdir -p /mnt/nfs-fix
sudo mount -t nfs 10.0.0.5:/share/kubernetes-pv /mnt/nfs-fix
sudo chown -R 1000:1000 /mnt/nfs-fix/dev-gitea-shared-storage-pvc-*
sudo chmod -R 755 /mnt/nfs-fix/dev-gitea-shared-storage-pvc-*
sudo umount /mnt/nfs-fix

# Restart pod
kubectl delete pod -n dev -l app.kubernetes.io/name=gitea
```

### Step 4: Verify Installation

```bash
# Check pod status (should be 1/1 Running)
kubectl get pods -n dev -l app.kubernetes.io/name=gitea

# Check logs
kubectl logs -n dev -l app.kubernetes.io/name=gitea --tail=50

# Look for successful startup messages:
# "Server Listen: http://0.0.0.0:3000"
# "Serving [v1] ...router was initialized"
```

---

## Verification

### Step 1: Local Port Forward Test

```bash
# Port forward to access Gitea locally
kubectl port-forward -n dev svc/gitea-http 3000:3000

# In browser, visit: http://localhost:3000
# Should see Gitea UI
```

### Step 2: Configure Cloudflare Tunnel

Add the Gitea subdomain to your Cloudflare tunnel:

1. **Go to Cloudflare Dashboard:**
   - https://one.dash.cloudflare.com/

2. **Navigate to Tunnel:**
   - Access → Tunnels → Select your tunnel
   - Click **Public Hostname** tab

3. **Add hostname:**
   - Click **Add a public hostname**
   - **Subdomain:** `git`
   - **Domain:** `jondepalma.net`
   - **Type:** HTTP
   - **URL:** `traefik.traefik.svc.cluster.local:80`
   - **Additional settings:**
     - Toggle **"No TLS Verify"** to ON
   - Click **Save**

4. **Wait for DNS propagation** (usually 1-2 minutes)

### Step 3: Access Gitea

```bash
# From outside your network (phone on cellular, laptop not on home WiFi):
# Visit: https://git.jondepalma.net

# Should see Gitea login page
```

### Step 4: Login

```bash
# Get admin credentials
echo "Username:"
kubectl get secret gitea-admin -n dev -o jsonpath='{.data.username}' | base64 --decode
echo ""

echo "Password:"
kubectl get secret gitea-admin -n dev -o jsonpath='{.data.password}' | base64 --decode
echo ""

# Login at: https://git.jondepalma.net
```

### Step 5: Create Test Repository

1. Click **+** button (top right) → **New Repository**
2. **Repository Name:** `test-repo`
3. **Description:** `Test repository`
4. **Initialize with README:** ✓
5. Click **Create Repository**

### Step 6: Test Git Operations

```bash
# Clone the test repo
git clone https://git.jondepalma.net/admin/test-repo.git
cd test-repo

# Make a change
echo "# Hello from Gitea!" >> README.md
git add README.md
git commit -m "Update README"

# Push (will prompt for username/password)
git push origin main

# Use your admin credentials when prompted
```

### Step 7: Generate Access Token (For CI/CD)

1. **User Settings** (top right avatar) → **Applications**
2. **Generate New Token**
   - Token Name: `ci-cd-token`
   - Select scopes: `repo`, `write:repository`
3. **Generate Token**
4. **Save the token** - you won't see it again!

---

## Troubleshooting

### Issue: CrashLoopBackOff - Permission Denied

**Symptoms:**
```
mkdir: can't create directory '/data/git/': Permission denied
```

**Solution:**
Run the NFS permissions fix (see [NFS Permissions Fix](#nfs-permissions-fix) section)

---

### Issue: Password Authentication Failed

**Symptoms:**
```
pq: password authentication failed for user "gitea"
```

**Causes:**
1. PostgreSQL user password doesn't match secret
2. Inline config secret not applied
3. Wrong password in inline config

**Solution:**

```bash
# Verify PostgreSQL password matches secret
GITEA_DB_PASSWORD=$(kubectl get secret gitea-external-db -n dev -o jsonpath='{.data.password}' | base64 --decode)
POSTGRES_PASS=$(kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 --decode)

# Update PostgreSQL with correct password
kubectl exec -it -n dev postgresql-dev-0 -- bash -c "PGPASSWORD='$POSTGRES_PASS' psql -U postgres -c \"ALTER USER gitea WITH PASSWORD '$GITEA_DB_PASSWORD';\""

# Test connection
kubectl run test-db --rm -it --image=postgres:15 -n dev --restart=Never -- \
  bash -c "PGPASSWORD='$GITEA_DB_PASSWORD' psql -h postgresql-dev.dev.svc.cluster.local -U gitea -d gitea -c 'SELECT version();'"

# If successful, restart Gitea
kubectl delete pod -n dev -l app.kubernetes.io/name=gitea
```

---

### Issue: "postgresschema" in Logs

**Symptoms:**
```
PING DATABASE postgresschema
```

**Cause:**
Helm chart concatenates `DB_TYPE` + `SCHEMA` fields

**Solution:**
Use inline config secret (already implemented in this guide)

---

### Issue: Can't Access via Domain

**Symptoms:**
- Local port-forward works
- External domain doesn't load

**Solution:**

```bash
# Check Cloudflare tunnel is running
kubectl get pods -n cloudflare
kubectl logs -n cloudflare -l app=cloudflared --tail=20

# Verify DNS
dig git.jondepalma.net
# Should show CNAME to Cloudflare tunnel

# Check ingress
kubectl get ingress -n dev

# Verify Traefik is working
kubectl get svc -n traefik
```

---

### Issue: Init Container Fails

**Check each init container:**

```bash
# Check which init container is failing
kubectl get pod -n dev -l app.kubernetes.io/name=gitea -o jsonpath='{range .items[0].status.initContainerStatuses[*]}{.name}{": "}{.state}{"\n"}{end}'

# Check logs for each
kubectl logs -n dev -l app.kubernetes.io/name=gitea -c init-directories --tail=50
kubectl logs -n dev -l app.kubernetes.io/name=gitea -c init-app-ini --tail=50
kubectl logs -n dev -l app.kubernetes.io/name=gitea -c configure-gitea --tail=50
```

---

## Maintenance

### Viewing Logs

```bash
# View main Gitea logs
kubectl logs -n dev -l app.kubernetes.io/name=gitea -f

# View init container logs
kubectl logs -n dev -l app.kubernetes.io/name=gitea -c configure-gitea --tail=100

# View all pod events
kubectl describe pod -n dev -l app.kubernetes.io/name=gitea
```

### Restarting Gitea

```bash
# Restart by deleting pod (will be recreated)
kubectl delete pod -n dev -l app.kubernetes.io/name=gitea

# Or restart via deployment
kubectl rollout restart deployment -n dev -l app.kubernetes.io/name=gitea
```

### Updating Gitea

```bash
# Update Helm repo
helm repo update

# Check for new versions
helm search repo gitea-charts/gitea

# Upgrade (preserves data and secrets)
helm upgrade gitea gitea-charts/gitea \
  --namespace dev \
  -f ~/pi-cluster-infrastructure/applications/gitea/values.yaml

# Watch rollout
kubectl get pods -n dev -w
```

### Backing Up Gitea Data

```bash
# Backup Gitea data from NFS
sudo mkdir -p /mnt/nfs-backup
sudo mount -t nfs 10.0.0.5:/share/kubernetes-pv /mnt/nfs-backup

# Create backup
BACKUP_DATE=$(date +%Y%m%d)
sudo tar -czf ~/gitea-backup-$BACKUP_DATE.tar.gz \
  /mnt/nfs-backup/dev-gitea-shared-storage-pvc-*

# Backup database
kubectl exec -n dev postgresql-dev-0 -- bash -c "PGPASSWORD='$POSTGRES_PASS' pg_dump -U postgres gitea" > ~/gitea-db-backup-$BACKUP_DATE.sql

# Copy to NAS
scp ~/gitea-backup-$BACKUP_DATE.tar.gz admin@10.0.0.5:/share/backups/
scp ~/gitea-db-backup-$BACKUP_DATE.sql admin@10.0.0.5:/share/backups/

# Cleanup
sudo umount /mnt/nfs-backup
rm ~/gitea-backup-$BACKUP_DATE.tar.gz ~/gitea-db-backup-$BACKUP_DATE.sql
```

### Changing Admin Password

```bash
# Option 1: Via Gitea UI
# Login → Settings → Account → Change Password

# Option 2: Via CLI
kubectl exec -n dev -l app.kubernetes.io/name=gitea -- \
  gitea admin user change-password -u admin -p 'new-password-here'

# Update secret to match (optional, for documentation)
kubectl delete secret gitea-admin -n dev
kubectl create secret generic gitea-admin \
  --from-literal=username=admin \
  --from-literal=password='new-password-here' \
  --from-literal=email=admin@jondepalma.net \
  -n dev
```

### Rotating Database Password

```bash
# Generate new password
NEW_DB_PASS=$(openssl rand -base64 32)

# Update PostgreSQL
POSTGRES_PASS=$(kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 --decode)
kubectl exec -it -n dev postgresql-dev-0 -- bash -c "PGPASSWORD='$POSTGRES_PASS' psql -U postgres -c \"ALTER USER gitea WITH PASSWORD '$NEW_DB_PASS';\""

# Update secrets
kubectl delete secret gitea-external-db -n dev
kubectl create secret generic gitea-external-db \
  --from-literal=password="$NEW_DB_PASS" \
  -n dev

kubectl delete secret gitea-inline-config -n dev
cat > /tmp/gitea-inline-config.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitea-inline-config
  namespace: dev
  labels:
    app.kubernetes.io/managed-by: Helm
  annotations:
    meta.helm.sh/release-name: gitea
    meta.helm.sh/release-namespace: dev
type: Opaque
stringData:
  database: |
    DB_TYPE=postgres
    HOST=postgresql-dev.dev.svc.cluster.local:5432
    NAME=gitea
    USER=gitea
    PASSWD=${NEW_DB_PASS}
    SSL_MODE=disable
EOF

kubectl apply -f /tmp/gitea-inline-config.yaml
rm /tmp/gitea-inline-config.yaml

# Restart Gitea
kubectl delete pod -n dev -l app.kubernetes.io/name=gitea
```

---

## Summary

You now have a fully functional Gitea installation with:

- ✅ **External PostgreSQL** - Using existing database server
- ✅ **Secure secrets** - Credentials in Kubernetes secrets, not hardcoded
- ✅ **NFS storage** - Persistent data on NAS
- ✅ **Working permissions** - Manual fix for NFS ownership
- ✅ **Inline config** - Workaround for Helm chart limitations
- ✅ **External access** - Via Cloudflare Tunnel
- ✅ **Production-ready** - Proper resource limits and security contexts

### Quick Reference Commands

```bash
# Check status
kubectl get pods -n dev -l app.kubernetes.io/name=gitea

# View logs
kubectl logs -n dev -l app.kubernetes.io/name=gitea -f

# Restart
kubectl delete pod -n dev -l app.kubernetes.io/name=gitea

# Fix permissions
~/fix-gitea-nfs-permissions.sh

# Get admin password
kubectl get secret gitea-admin -n dev -o jsonpath='{.data.password}' | base64 --decode

# Access
https://git.jondepalma.net
```

---

## Files to Commit to Git

These files should be in your infrastructure repo:

```
~/pi-cluster-infrastructure/
└── applications/
    └── gitea/
        ├── values.yaml              # Helm values (NO passwords)
        └── README.md                # This guide
```

**DO NOT commit:**
- Inline config secret (has password)
- Any files with actual passwords
- PVC definitions (auto-created)

---

**Last Updated:** 2024-11-19  
**Chart Version:** gitea-12.4.0  
**App Version:** 1.24.6
