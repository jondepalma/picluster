# Ghost Blog Quick Reference

## Quick Status Check

```bash
# Check everything
kubectl get all -n ghost
kubectl get all -n prod | grep mysql-ghost
kubectl get pvc -n ghost
kubectl get pvc -n prod | grep mysql-ghost

# Check Ghost logs
kubectl logs -l app=ghost -n ghost --tail=50 -f

# Check MySQL logs
kubectl logs -l app=mysql-ghost -n prod --tail=50 -f
```

## Common Operations

### Access Ghost Admin
```
URL: https://blog.yourdomain.com/ghost
```

### Restart Ghost
```bash
kubectl rollout restart deployment/ghost -n ghost
```

### Restart MySQL
```bash
kubectl rollout restart statefulset/mysql-ghost -n prod
```

### Get Ghost Pod Name
```bash
kubectl get pod -l app=ghost -n ghost
```

### Shell into Ghost Pod
```bash
GHOST_POD=$(kubectl get pod -l app=ghost -n ghost -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it -n ghost $GHOST_POD -- sh
```

### Shell into MySQL Pod
```bash
kubectl exec -it -n prod mysql-ghost-0 -- bash
```

### MySQL Command Line
```bash
# Get MySQL password first
MYSQL_PASS=$(kubectl get secret mysql-ghost -n prod -o jsonpath='{.data.password}' | base64 -d)

# Connect to MySQL
kubectl exec -it -n prod mysql-ghost-0 -- \
  mysql -u ghost -p${MYSQL_PASS} ghost
```

### Quick Backup
```bash
# Database backup
MYSQL_ROOT=$(kubectl get secret mysql-ghost -n prod -o jsonpath='{.data.root-password}' | base64 -d)
kubectl exec -n prod mysql-ghost-0 -- \
  mysqldump -u root -p${MYSQL_ROOT} ghost | \
  gzip > ghost-backup-$(date +%Y%m%d).sql.gz

# Content backup
GHOST_POD=$(kubectl get pod -l app=ghost -n ghost -o jsonpath='{.items[0].metadata.name}')
kubectl cp ghost/${GHOST_POD}:/var/lib/ghost/content ./ghost-content-backup
```

### Scale Ghost
```bash
# Scale up to 2 replicas
kubectl scale deployment ghost -n ghost --replicas=2

# Scale down to 1 replica
kubectl scale deployment ghost -n ghost --replicas=1
```

### View Resource Usage
```bash
kubectl top pod -l app=ghost -n ghost
kubectl top pod -l app=mysql-ghost -n prod
```

### Port Forward for Testing
```bash
# Access Ghost locally without ingress
kubectl port-forward -n ghost svc/ghost 8080:2368
# Then open: http://localhost:8080

# Access MySQL locally
kubectl port-forward -n prod svc/mysql-ghost 3306:3306
# Then connect with: mysql -h 127.0.0.1 -u ghost -p
```

## Troubleshooting Commands

### Check DNS Resolution
```bash
GHOST_POD=$(kubectl get pod -l app=ghost -n ghost -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n ghost $GHOST_POD -- nslookup mysql-ghost.prod.svc.cluster.local
```

### Test MySQL Connection
```bash
GHOST_POD=$(kubectl get pod -l app=ghost -n ghost -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n ghost $GHOST_POD -- nc -zv mysql-ghost.prod.svc.cluster.local 3306
```

### Check Events
```bash
kubectl get events -n ghost --sort-by='.lastTimestamp'
kubectl get events -n prod --sort-by='.lastTimestamp' | grep mysql
```

### View Pod Details
```bash
kubectl describe pod -l app=ghost -n ghost
kubectl describe pod mysql-ghost-0 -n prod
```

### Check ConfigMap
```bash
kubectl get configmap mysql-ghost-config -n prod -o yaml
```

## Secret Management

### View Secret Keys (not values)
```bash
kubectl get secret ghost-secrets -n ghost -o jsonpath='{.data}' | jq 'keys'
kubectl get secret mysql-ghost -n prod -o jsonpath='{.data}' | jq 'keys'
```

### Decode Secret Value (use carefully!)
```bash
# MySQL Ghost password
kubectl get secret mysql-ghost -n prod -o jsonpath='{.data.password}' | base64 -d

# MySQL Root password
kubectl get secret mysql-ghost -n prod -o jsonpath='{.data.root-password}' | base64 -d

# Ghost DB password
kubectl get secret ghost-secrets -n ghost -o jsonpath='{.data.db-password}' | base64 -d
```

## Update Operations

### Update Ghost Image
```bash
# Edit deployment file with new version
kubectl set image deployment/ghost ghost=ghost:5.131.0 -n ghost

# Or edit directly
kubectl edit deployment ghost -n ghost

# Watch rollout
kubectl rollout status deployment/ghost -n ghost

# Check history
kubectl rollout history deployment/ghost -n ghost

# Rollback if needed
kubectl rollout undo deployment/ghost -n ghost
```

### Update MySQL Configuration
```bash
# Edit ConfigMap
kubectl edit configmap mysql-ghost-config -n prod

# Restart MySQL to apply changes
kubectl rollout restart statefulset/mysql-ghost -n prod
```

## Cleanup Commands

### Delete Ghost (keep data)
```bash
kubectl delete deployment ghost -n ghost
kubectl delete service ghost -n ghost
kubectl delete ingress ghost -n ghost
# PVC and secrets remain
```

### Delete MySQL (keep data)
```bash
kubectl delete statefulset mysql-ghost -n prod
kubectl delete service mysql-ghost -n prod
# PVC and secrets remain
```

### Complete Removal (INCLUDING DATA!)
```bash
# WARNING: This deletes everything including data!
kubectl delete namespace ghost
kubectl delete statefulset mysql-ghost -n prod
kubectl delete service mysql-ghost -n prod
kubectl delete configmap mysql-ghost-config -n prod
kubectl delete pvc -l app=mysql-ghost -n prod
kubectl delete secret mysql-ghost -n prod
```

## Performance Tuning

### Increase MySQL Resources
```bash
# Edit 03-mysql-statefulset.yaml, change:
# resources.requests.memory: "1Gi"
# resources.requests.cpu: "500m"
# Then apply:
kubectl apply -f 03-mysql-statefulset.yaml
kubectl rollout restart statefulset/mysql-ghost -n prod
```

### Increase Ghost Resources
```bash
# Edit 05-ghost-deployment.yaml, change:
# resources.requests.memory: "512Mi"
# resources.requests.cpu: "200m"
# Then apply:
kubectl apply -f 05-ghost-deployment.yaml
```

## Monitoring

### Watch Logs in Real-time
```bash
# Ghost logs
kubectl logs -l app=ghost -n ghost -f

# MySQL logs
kubectl logs -l app=mysql-ghost -n prod -f

# Both simultaneously (requires tmux or multiple terminals)
```

### Check MySQL Slow Queries
```bash
kubectl exec -n prod mysql-ghost-0 -- tail -f /var/lib/mysql/slow-query.log
```

### Get Ghost Metrics
```bash
GHOST_POD=$(kubectl get pod -l app=ghost -n ghost -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n ghost $GHOST_POD -- wget -qO- http://localhost:2368/ghost/api/v4/admin/site/
```

## Emergency Recovery

### Restore from Backup
```bash
# Restore MySQL database
gunzip < ghost-backup-20251122.sql.gz | \
  kubectl exec -i -n prod mysql-ghost-0 -- \
  mysql -u root -p${MYSQL_ROOT} ghost

# Restore Ghost content
kubectl cp ./ghost-content-backup ${GHOST_POD}:/var/lib/ghost/content -n ghost
kubectl rollout restart deployment/ghost -n ghost
```

### Fix Corrupted MySQL
```bash
kubectl exec -it -n prod mysql-ghost-0 -- mysqlcheck -u root -p${MYSQL_ROOT} --auto-repair --all-databases
```

---

**Quick Deploy (after secrets created):**
```bash
kubectl apply -f 00-ghost-namespace.yaml && \
kubectl apply -f 03-mysql-statefulset.yaml && \
sleep 30 && \
kubectl apply -f 04-ghost-pvc.yaml && \
kubectl apply -f 05-ghost-deployment.yaml && \
kubectl apply -f 06-ghost-service.yaml && \
kubectl apply -f 07-ghost-ingress.yaml
```

**Quick Check:**
```bash
kubectl get pod -n ghost && kubectl get pod -n prod | grep mysql
```
