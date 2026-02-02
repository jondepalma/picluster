# Ghost Blog Production Deployment Files

Complete production-ready Ghost blog deployment for Kubernetes with MySQL database.

## 📁 Files Overview

### Deployment Files (in order)

1. **00-ghost-namespace.yaml** - Ghost namespace creation
2. **03-mysql-statefulset.yaml** - MySQL database (in `prod` namespace)
3. **04-ghost-pvc.yaml** - Ghost content storage
4. **05-ghost-deployment.yaml** - Ghost application
5. **06-ghost-service.yaml** - Ghost service
6. **07-ghost-ingress.yaml** - Ghost ingress with TLS

### Secret Templates (DO NOT APPLY DIRECTLY)

- **01-mysql-secret-template.yaml** - MySQL secret template
- **02-ghost-secret-template.yaml** - Ghost secret template

These are templates only. Create actual secrets using the commands in DEPLOYMENT-GUIDE.md

### Documentation

- **DEPLOYMENT-GUIDE.md** - Complete step-by-step deployment guide
- **QUICK-REFERENCE.md** - Command reference for common operations
- **README.md** - This file

## 🚀 Quick Start

### Prerequisites
- Kubernetes cluster with `nfs-client` StorageClass
- `prod` namespace exists
- kubectl configured and working

### Deploy in 3 Steps

#### 1. Create Secrets (REQUIRED FIRST)

```bash
# Generate passwords
MYSQL_ROOT=$(openssl rand -base64 32)
MYSQL_GHOST=$(openssl rand -base64 32)

# Create MySQL secret (prod namespace)
kubectl create secret generic mysql-ghost \
  --from-literal=root-password="${MYSQL_ROOT}" \
  --from-literal=password="${MYSQL_GHOST}" \
  -n prod

# Create Ghost secret (ghost namespace - will be created in step 2)
kubectl create secret generic ghost-secrets \
  --from-literal=db-password="${MYSQL_GHOST}" \
  -n ghost

# SAVE THESE PASSWORDS!
echo "MySQL Root: ${MYSQL_ROOT}"
echo "MySQL Ghost: ${MYSQL_GHOST}"
```

#### 2. Update Configuration

Edit these files and replace `blog.yourdomain.com` with your actual domain:
- `05-ghost-deployment.yaml` (line ~60)
- `07-ghost-ingress.yaml` (TLS hosts and rules)

#### 3. Deploy

```bash
# Deploy all components
kubectl apply -f 00-ghost-namespace.yaml
kubectl apply -f 03-mysql-statefulset.yaml

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app=mysql-ghost -n prod --timeout=300s

# Deploy Ghost
kubectl apply -f 04-ghost-pvc.yaml
kubectl apply -f 05-ghost-deployment.yaml
kubectl apply -f 06-ghost-service.yaml
kubectl apply -f 07-ghost-ingress.yaml

# Check status
kubectl get all -n ghost
kubectl get all -n prod | grep mysql
```

#### 4. Access Ghost

Navigate to: `https://blog.yourdomain.com/ghost`

Create your admin account on first visit.

## 🏗️ Architecture

```
Ghost Namespace (ghost)
├── Deployment: ghost (1 replica)
│   └── Image: ghost:5.130.5
│   └── Resources: 256Mi RAM, 100m CPU
├── Service: ghost:2368
├── PVC: ghost-content (10Gi)
└── Ingress: blog.yourdomain.com

Prod Namespace (prod)  
└── StatefulSet: mysql-ghost
    ├── Image: mysql:8.0
    ├── Resources: 512Mi RAM, 250m CPU
    └── PVC: 20Gi (per replica)

Secrets:
├── prod/mysql-ghost (root-password, password)
└── ghost/ghost-secrets (db-password)
```

## 📊 Resource Requirements

### Minimum
- **CPU**: 350m (100m Ghost + 250m MySQL)
- **RAM**: 768Mi (256Mi Ghost + 512Mi MySQL)
- **Storage**: 30Gi (10Gi Ghost + 20Gi MySQL)

### Recommended
- **CPU**: 1.5 cores (500m Ghost + 1000m MySQL)
- **RAM**: 2Gi (512Mi Ghost + 1.5Gi MySQL)
- **Storage**: 50Gi+ for growth

## 🔒 Security Notes

1. **Never commit secrets to Git** - Templates only!
2. **Store passwords in password manager** - You'll need them
3. **Use strong, random passwords** - 32+ characters
4. **Rotate secrets every 6 months** - Set a reminder
5. **Backup encrypted** - Use `gpg -c` for secret backups

## 📚 Documentation

- **Full Deployment Guide**: See `DEPLOYMENT-GUIDE.md` for detailed instructions
- **Quick Reference**: See `QUICK-REFERENCE.md` for common commands
- **Troubleshooting**: Both docs include troubleshooting sections

## ⚙️ Common Operations

```bash
# Restart Ghost
kubectl rollout restart deployment/ghost -n ghost

# View logs
kubectl logs -l app=ghost -n ghost -f

# Scale Ghost
kubectl scale deployment ghost -n ghost --replicas=2

# Backup database
MYSQL_ROOT=$(kubectl get secret mysql-ghost -n prod -o jsonpath='{.data.root-password}' | base64 -d)
kubectl exec -n prod mysql-ghost-0 -- \
  mysqldump -u root -p${MYSQL_ROOT} ghost > ghost-backup.sql

# Port forward for testing
kubectl port-forward -n ghost svc/ghost 8080:2368
```

## 🆘 Need Help?

1. Check `DEPLOYMENT-GUIDE.md` for step-by-step instructions
2. Check `QUICK-REFERENCE.md` for common commands
3. Review troubleshooting sections in both docs
4. Check pod logs: `kubectl logs -l app=ghost -n ghost`
5. Check events: `kubectl get events -n ghost --sort-by='.lastTimestamp'`

## 🔄 Updates

To update Ghost version:

```bash
# Edit 05-ghost-deployment.yaml
# Change: image: ghost:5.130.5
# To: image: ghost:5.131.0 (or latest)

# Apply update
kubectl apply -f 05-ghost-deployment.yaml

# Monitor rollout
kubectl rollout status deployment/ghost -n ghost
```

## 🗑️ Cleanup

**Warning: This deletes everything including data!**

```bash
# Delete Ghost
kubectl delete namespace ghost

# Delete MySQL (careful!)
kubectl delete statefulset mysql-ghost -n prod
kubectl delete service mysql-ghost -n prod
kubectl delete pvc -l app=mysql-ghost -n prod
kubectl delete configmap mysql-ghost-config -n prod
kubectl delete secret mysql-ghost -n prod
```

## 📝 File Checklist

Before deployment, verify you have:

- [ ] Updated domain in `05-ghost-deployment.yaml`
- [ ] Updated domain in `07-ghost-ingress.yaml`
- [ ] Created MySQL secret in `prod` namespace
- [ ] Created Ghost secret in `ghost` namespace
- [ ] Saved all passwords in password manager
- [ ] Verified `nfs-client` StorageClass exists
- [ ] Verified `prod` namespace exists

## 🎯 What's Different from the Original YAML?

1. ✅ **Separate namespace for MySQL** (`prod` vs `ghost`)
2. ✅ **Production secrets management** (proper templates)
3. ✅ **Specific Ghost version** (5.130.5 instead of tag)
4. ✅ **Debian-based image** (matches your Pi cluster OS)
5. ✅ **StatefulSet for MySQL** (proper database deployment)
6. ✅ **Resource limits** (prevents resource exhaustion)
7. ✅ **Health checks** (liveness and readiness probes)
8. ✅ **Optimized MySQL config** (tuned for Pi cluster)
9. ✅ **TLS/Ingress ready** (with security headers)
10. ✅ **Cross-namespace connection** (Ghost → MySQL)
11. ✅ **Init container** (waits for MySQL before starting)
12. ✅ **Complete documentation** (deployment and operations)

## 📌 Important Notes

- **MySQL is in `prod` namespace**, Ghost is in `ghost` namespace
- **Database connection** uses full DNS: `mysql-ghost.prod.svc.cluster.local`
- **Secrets must match** - Ghost's db-password = MySQL's password
- **No mail configured** by default (add later if needed)
- **Uses SQLite by default** until MySQL is configured
- **First admin account** created on first visit to /ghost

---

**Version**: 1.0  
**Ghost Version**: 5.130.5  
**MySQL Version**: 8.0  
**Last Updated**: November 2025
