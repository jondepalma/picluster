# Storage Strategy: 3-Node Pi Cluster with NVMe

**Optimized for: Performance databases on NVMe, bulk storage on NAS**

---

## Hardware Overview

### Cluster Composition
- **3x Raspberry Pi 5** (8GB RAM each = 24GB total)
- **Control Plane (pi-control):**
  - 256GB SD card (OS, logs, temp)
  - 1TB M.2 NVMe SSD (high-performance workloads)
- **Workers (pi-worker-1, pi-worker-2):**
  - 256GB SD card each (OS, logs, temp)
- **QNAP NAS:**
  - Network storage (500GB+ allocated)
  - Backup destination

---

## Storage Tiers

### Tier 1: NVMe SSD (1TB on pi-control)
**Performance:** ~3000 MB/s read, ~2000 MB/s write, <1ms latency

**Best for:**
- PostgreSQL databases (dev & prod)
- ScyllaDB clusters (dev & prod)
- Redis cache/persistence
- Build artifacts and CI/CD cache
- etcd cluster state
- Any workload requiring low latency and high IOPS

**Kubernetes Storage Class:** `local-path-nvme`

**Key Advantage:** 30-50x faster than SD cards, 20-30x faster than NAS over gigabit

### Tier 2: QNAP NAS via NFS (500GB+)
**Performance:** ~110 MB/s read, ~90 MB/s write, ~5ms latency

**Best for:**
- Git repositories (large, infrequently written)
- Container registries (large images)
- Backup storage
- Application file storage
- Prometheus metrics (historical data)
- Ghost blog content
- Any data that needs to survive node failures

**Kubernetes Storage Classes:** `nfs-client`, `nfs-db`

**Key Advantage:** Shared across all nodes, survives cluster rebuilds, easy to back up

### Tier 3: SD Cards (256GB × 3 = 768GB total)
**Performance:** ~90 MB/s read, ~50 MB/s write, ~10ms latency

**Best for:**
- Operating system
- System logs (rotated frequently)
- Temporary files
- Pod ephemeral storage
- Non-critical caches

**Kubernetes Storage Class:** `local-path` (default)

**Key Advantage:** Built-in, no configuration needed, adequate for OS and logs

---

## Why This Setup Works

### NVMe on Control Plane = Smart Architecture

**Traditional approach:** Run databases on worker nodes
**Problem:** SD cards bottleneck database performance

**This approach:** Run databases on control plane with NVMe
**Benefits:**
1. **Performance:** Databases get NVMe-level performance
2. **Stability:** Control plane typically has lighter workload than workers
3. **Centralization:** Single high-performance storage location
4. **Cost-effective:** Only one NVMe needed instead of three

### Node Tainting Strategy

**Control plane is tainted by default:**
```bash
--node-taint CriticalAddonsOnly=true:NoExecute
```

**This means:** Only system pods run on control plane unless you explicitly allow workloads

**For databases on NVMe, add tolerations:**
```yaml
tolerations:
- key: CriticalAddonsOnly
  operator: Equal
  value: "true"
  effect: NoExecute

nodeSelector:
  kubernetes.io/hostname: pi-control
```

This ensures database pods:
1. Can run on control plane (toleration)
2. MUST run on pi-control (nodeSelector)
3. Get access to NVMe storage

---

## Storage Allocation Strategy

### Development Environment (100GB total)
- PostgreSQL: 30GB (NVMe)
- ScyllaDB: 30GB (NVMe)
- Build cache: 20GB (NVMe)
- Gitea repos: 50GB (NAS)

### Production Environment (300GB total)
- PostgreSQL: 100GB (NVMe)
- ScyllaDB: 150GB (NVMe)
- Redis: 10GB (NVMe)
- Ghost blog: 10GB (NAS)
- Application data: 50GB (NAS)
- Backups: 100GB (NAS)

### Monitoring & Operations (60GB total)
- Prometheus: 50GB (NAS)
- Grafana: 5GB (NAS)
- Logs: 30GB (SD cards, distributed)

### Reserved Capacity
- NVMe: 570GB available for growth
- NAS: 135GB available
- SD cards: 618GB available across nodes

---

## Performance Comparison

### Database Write Performance (PostgreSQL)

| Storage Type | Transactions/sec | Latency (avg) | 95th Percentile |
|--------------|------------------|---------------|-----------------|
| **NVMe SSD** | ~8,000 TPS | 0.5ms | 1.2ms |
| **NAS/NFS** | ~800 TPS | 5ms | 12ms |
| **SD Card** | ~200 TPS | 20ms | 50ms |

**Real-world impact:** NVMe is 10x faster than NAS, 40x faster than SD for databases

### Application Deployment Time

| Storage Type | Build Time | Deploy Time | Total |
|--------------|-----------|-------------|-------|
| **NVMe** | 30s | 5s | 35s |
| **NAS** | 45s | 10s | 55s |
| **SD Card** | 90s | 15s | 105s |

---

## Storage Class Deployment Examples

### Example 1: PostgreSQL on NVMe

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: prod
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path-nvme
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: prod
spec:
  serviceName: postgresql
  replicas: 1
  template:
    spec:
      # Force scheduling on pi-control
      nodeSelector:
        kubernetes.io/hostname: pi-control
      # Allow running on tainted control plane
      tolerations:
      - key: CriticalAddonsOnly
        operator: Equal
        value: "true"
        effect: NoExecute
      containers:
      - name: postgres
        image: postgres:15-alpine
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgres-data
```

### Example 2: Gitea on NAS

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitea-repos
  namespace: dev
spec:
  accessModes:
    - ReadWriteMany  # Can be shared if needed
  storageClassName: nfs-client
  resources:
    requests:
      storage: 50Gi
```

### Example 3: Application Logs (Ephemeral)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-logs
  namespace: prod
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path  # Default, uses SD
  resources:
    requests:
      storage: 10Gi
```

---

## Backup Strategy

### NVMe Backups (Critical Data)

**Frequency:** Daily at 2 AM

**Method:** Database dumps to NAS

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: prod
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          nodeSelector:
            kubernetes.io/hostname: pi-control
          containers:
          - name: backup
            image: postgres:15-alpine
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h postgresql -U postgres -d proddb | gzip > /backups/postgres-$(date +%Y%m%d).sql.gz
              find /backups -name "postgres-*.sql.gz" -mtime +7 -delete
            volumeMounts:
            - name: backups
              mountPath: /backups
          volumes:
          - name: backups
            nfs:
              server: 10.0.0.5
              path: /share/kubernetes-backups
          restartPolicy: OnFailure
```

### NAS Backups (Already Persistent)

**Frequency:** Weekly snapshots on QNAP

**QNAP Configuration:**
1. Control Panel → Backup & Restore
2. Snapshot Vault
3. Create policy:
   - Schedule: Weekly (Sunday 2 AM)
   - Retention: 4 weeks
   - Folders: kubernetes-pv, kubernetes-backups

---

## Failure Scenarios & Recovery

### Scenario 1: NVMe Failure

**Impact:** Databases on pi-control unavailable

**Recovery:**
1. Replace NVMe SSD
2. Format new SSD
3. Restore from NAS backups
4. Downtime: ~30 minutes

**Mitigation:** Daily backups, consider PostgreSQL replication to NAS

### Scenario 2: NAS Failure

**Impact:** Git repos, backups, bulk storage unavailable

**Recovery:**
1. Fix NAS or restore from off-site backup
2. Databases continue running (on NVMe)
3. Critical services unaffected

**Mitigation:** QNAP RAID configuration, cloud backups

### Scenario 3: SD Card Failure

**Impact:** Single node unavailable

**Recovery:**
1. Flash new SD card
2. Rejoin node to cluster
3. Pods reschedule automatically
4. Downtime: ~15 minutes for that node

**Mitigation:** Regular SD card replacement, no critical data on SD

### Scenario 4: pi-control Node Failure

**Impact:** Databases and control plane unavailable

**Recovery:**
1. Fix pi-control hardware
2. If unfixable, move NVMe to new Pi
3. Restore cluster state from backup
4. Downtime: 1-4 hours

**Mitigation:** 
- Regular etcd backups
- Database backups to NAS
- Document recovery procedures
- Consider HA control plane (requires 3+ control nodes, not practical here)

---

## Monitoring Storage Health

### NVMe SMART Monitoring

```bash
# Install smartmontools
sudo apt install smartmontools

# Check NVMe health
sudo smartctl -a /dev/nvme0n1

# Key metrics to watch:
# - Percentage Used
# - Available Spare
# - Critical Warning
```

### Kubernetes Storage Monitoring

```bash
# Check PVC usage
kubectl get pvc --all-namespaces

# Check available storage
kubectl exec -it <pod> -- df -h

# Check I/O stats
kubectl top pods --all-namespaces
```

### Grafana Dashboards

Import these dashboards:
- **Node Exporter Full** (ID: 1860) - Disk I/O, SMART data
- **Kubernetes Persistent Volumes** (ID: 13646) - PVC usage
- **PostgreSQL Database** (ID: 9628) - Database performance

---

## Best Practices

### ✅ DO

1. **Use NVMe for databases** - Always put PostgreSQL, ScyllaDB, Redis on NVMe
2. **Use NAS for archives** - Git repos, backups, historical data
3. **Use SD for ephemeral** - Logs, temp files, caches
4. **Monitor NVMe health** - Run SMART checks monthly
5. **Test backups** - Restore from backup quarterly
6. **Keep SD cards fresh** - Replace annually

### ❌ DON'T

1. **Don't run databases on SD cards** - Too slow, will wear out
2. **Don't store critical data on SD** - Can fail without warning
3. **Don't fill NVMe completely** - Leave 20% free for performance
4. **Don't skip backups** - NVMe is single point of failure
5. **Don't over-provision NAS** - Network bandwidth is limited
6. **Don't ignore SMART warnings** - Replace failing drives immediately

---

## Scaling Considerations

### Adding More Workers (Future)

**New pi-worker-3:**
- 256GB SD card
- Runs application pods
- Uses NAS for persistent storage
- No need for additional NVMe (databases stay on pi-control)

### Upgrading NVMe

**If 1TB becomes insufficient:**
1. Back up all data to NAS
2. Replace 1TB NVMe with 2TB
3. Restore data
4. Update storage allocations

**Alternative:** Add NVMe to worker nodes for horizontal database scaling

### When to Add More NVMe

**Indicators:**
- NVMe >80% full
- Database I/O wait time increasing
- Need for database read replicas
- Multiple high-I/O applications

**Solution:** Add NVMe to one worker, run database replicas there

---

## Cost Analysis

### Storage Cost Breakdown

| Component | Capacity | Cost | $/GB | Notes |
|-----------|----------|------|------|-------|
| **NVMe SSD** | 1TB | $80 | $0.08 | Samsung 970 EVO Plus |
| **SD Cards** | 768GB | $150 | $0.20 | 3× Samsung EVO Select 256GB |
| **QNAP NAS** | 4TB | $600 | $0.15 | TS-473 with 4× 1TB drives |
| **M.2 HAT** | - | $25 | - | For Pi 5 |

**Total Storage:** ~5.8TB for ~$855 ($0.15/GB average)

**Compared to AWS:**
- EBS gp3: $0.08/GB/month = $464/month for 5.8TB
- **Break-even:** ~2 months

---

## Summary

Your 3-node cluster with NVMe on the control plane provides:

✅ **High-performance databases** - 30-50x faster than SD cards
✅ **Cost-effective** - One NVMe instead of three
✅ **Resilient** - Three storage tiers for different use cases
✅ **Scalable** - 570GB+ NVMe available for growth
✅ **Production-ready** - Proper backup and recovery strategies

**Key Insight:** By placing high-performance storage on the control plane (which typically has spare capacity), you get enterprise-grade database performance without needing NVMe on every node.

This is a smart, practical architecture for a homelab that balances performance, cost, and reliability.
