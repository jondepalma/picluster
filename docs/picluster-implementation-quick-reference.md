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

## Quick Reference

### Essential Commands

**Check cluster status:**
```bash
kubectl get nodes
kubectl get pods --all-namespaces
kubectl top nodes
```

**View logs:**
```bash
kubectl logs -f <pod-name> -n <namespace>
```

**Port forward for local testing:**
```bash
kubectl port-forward svc/ghost 2368:2368 -n prod
```

**Restart deployment:**
```bash
kubectl rollout restart deployment/<name> -n <namespace>
```

**Check ingress:**
```bash
kubectl get ingress --all-namespaces
```

### Service URLs

| Service | Internal URL | External URL |
|---------|-------------|--------------|
| Ghost Blog | http://ghost.prod.svc.cluster.local:2368 | https://blog.yourdomain.com |
| PostgreSQL (Dev) | postgresql-dev.dev.svc.cluster.local:5432 | N/A |
| PostgreSQL (Prod) | postgresql-prod-primary.prod.svc.cluster.local:5432 | N/A |
| ScyllaDB (Dev) | scylla-dev-client.dev.svc.cluster.local:9042 | N/A |
| ScyllaDB (Prod) | scylla-prod-client.prod.svc.cluster.local:9042 | N/A |
| Redis | redis-master.prod.svc.cluster.local:6379 | N/A |
| Gitea | http://gitea-http.dev.svc.cluster.local:3000 | https://git.yourdomain.com |
| Grafana | http://kube-prometheus-stack-grafana.monitoring.svc.cluster.local | https://grafana.yourdomain.com |

### Database Connection Strings

**PostgreSQL (Dev):**
```
postgresql://postgres:devpassword@postgresql-dev.dev.svc.cluster.local:5432/devdb
```

**PostgreSQL (Prod):**
```
postgresql://postgres:prodpassword@postgresql-prod-primary.prod.svc.cluster.local:5432/proddb
```

**ScyllaDB (Dev):**
```
Host: scylla-dev-client.dev.svc.cluster.local
Port: 9042
```

**Redis:**
```
redis://redis-master.prod.svc.cluster.local:6379
Password: redispassword
```

### Troubleshooting

**Pod won't start:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

**Service not accessible:**
```bash
kubectl get svc -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
```

**NFS mount issues:**
```bash
# On node
sudo mount -t nfs 10.0.0.5:/share/kubernetes-pv /mnt/test
df -h
```

**Check Cloudflare tunnel:**
```bash
sudo systemctl status cloudflared
sudo journalctl -u cloudflared -f
```

---

## Resource Allocation Summary

### Per Environment

| Environment | CPU | RAM | Storage |
|------------|-----|-----|---------|
| Development | 2 cores | 4GB | 100GB |
| Production | 3 cores | 6GB | 200GB |
| Monitoring | 1 core | 2GB | 60GB |
| Infrastructure | 1 core | 2GB | 20GB |
| **Total Used** | **7 cores** | **14GB** | **380GB** |
| **Available (3 nodes)** | **5 cores** | **10GB** | **388GB** |
| **Total Cluster** | **12 cores** | **24GB** | **768GB SD + 1TB NVMe** |

### Storage Distribution

**Local NVMe (pi-control - 1TB):**
- K3s etcd data: 10GB
- High-performance databases: 200GB
- Container image cache: 50GB
- Build artifacts: 50GB
- Reserved for performance-critical workloads: 690GB

**Local SD Cards (768GB total across 3 nodes):**
- OS and system: 60GB (20GB per node)
- Logs and temporary data: 60GB
- Local path provisioner: 100GB
- Reserved: 548GB

**NFS Storage (QNAP NAS):**

| Service | Storage Size | Type | Backup | Notes |
|---------|--------------|------|--------|-------|
| **Gitea Repos** | 50 GB | PV-NFS | Daily | Git repositories |
| **Ghost Blog** | 10 GB | PV-NFS | Weekly | Blog content |
| **Harbor Registry** | 100 GB | PV-NFS | Weekly | Container images (optional) |
| **Prometheus Data** | 50 GB | PV-NFS | Weekly | 15-day metrics |
| **Grafana** | 5 GB | PV-NFS | Weekly | Dashboards |
| **Backups** | 100 GB | PV-NFS | N/A | Backup destination |
| **Production Data** | 50 GB | PV-NFS | Daily | App data archives |
| **Total NAS** | **365 GB** | - | - | Bulk storage |

**NVMe Storage (pi-control - 1TB):**

| Service | Storage Size | Type | Location | Notes |
|---------|--------------|------|----------|-------|
| **PostgreSQL Dev** | 30 GB | PV-NVMe | pi-control | Development DB |
| **PostgreSQL Prod** | 100 GB | PV-NVMe | pi-control | Production DB (primary) |
| **ScyllaDB Dev** | 30 GB | PV-NVMe | pi-control | Development NoSQL |
| **ScyllaDB Prod** | 150 GB | PV-NVMe | pi-control | Production NoSQL |
| **Redis** | 10 GB | PV-NVMe | pi-control | Cache + persistence |
| **Build Cache** | 50 GB | PV-NVMe | pi-control | CI/CD artifacts |
| **K3s etcd** | 10 GB | Local | pi-control | Cluster state |
| **Container Cache** | 50 GB | Local | pi-control | Image layers |
| **Reserved** | 570 GB | - | pi-control | Future expansion |
| **Total NVMe** | **1000 GB** | - | - | High-performance |

**SD Card Storage (768GB total):**

| Node | Total | OS | Logs | Temp | Available |
|------|-------|-----|------|------|-----------|
| pi-control | 256 GB | 20 GB | 10 GB | 20 GB | 206 GB |
| pi-worker-1 | 256 GB | 20 GB | 10 GB | 20 GB | 206 GB |
| pi-worker-2 | 256 GB | 20 GB | 10 GB | 20 GB | 206 GB |
| **Total** | **768 GB** | **60 GB** | **30 GB** | **60 GB** | **618 GB** |

### Storage Performance Characteristics

| Storage Type | Read Speed | Write Speed | Latency | Use Case |
|--------------|-----------|-------------|---------|----------|
| **NVMe SSD** | ~3000 MB/s | ~2000 MB/s | < 1ms | Databases, high I/O |
| **SD Card** | ~90 MB/s | ~50 MB/s | ~10ms | OS, logs, temp files |
| **NFS (NAS)** | ~110 MB/s | ~90 MB/s | ~5ms | Bulk storage, backups |

### Storage Class Selection Guide

| Workload | Storage Class | Reason |
|----------|--------------|--------|
| **PostgreSQL** | local-path-nvme | Needs low latency, high IOPS |
| **ScyllaDB** | local-path-nvme | Optimized for NVMe performance |
| **Redis** | local-path-nvme | In-memory with persistence backup |
| **Git repositories** | nfs-client | Large files, bulk storage |
| **Container images** | nfs-client | Large, infrequently accessed |
| **Backups** | nfs-client | Archival, off-node storage |
| **Logs** | local-path | Temporary, can be lost |
| **Monitoring** | nfs-client | Historical data retention |
| **Application data** | nfs-db | Shared between pods |

| Service | Storage | Location |
|---------|---------|----------|
| PostgreSQL (Dev) | 30GB | NFS |
| PostgreSQL (Prod) | 100GB | NFS |
| ScyllaDB (Dev) | 30GB | NFS |
| ScyllaDB (Prod) | 90GB | NFS |
| Ghost Blog | 10GB | NFS |
| Gitea | 50GB | NFS |
| Redis | 10GB | NFS |
| Prometheus | 50GB | NFS |
| Grafana | 5GB | NFS |
| Backups | 100GB | NFS |
| **Total** | **475GB** | - |

---

## Next Steps

1. **Secure production:**
   - Change all default passwords
   - Setup proper SSL certificates
   - Configure firewall rules
   - Enable audit logging

2. **Optimize performance:**
   - Tune database parameters
   - Configure resource limits
   - Setup horizontal pod autoscaling
   - Optimize container images

3. **Expand functionality:**
   - Add more applications
   - Setup email notifications
   - Configure alerting rules
   - Implement log aggregation

4. **Documentation:**
   - Document your applications
   - Create runbooks
   - Setup wiki in Gitea
   - Document recovery procedures

---

## Conclusion

This simplified architecture provides:
- ✅ Public website with Ghost blog
- ✅ Complete development environment
- ✅ Production-ready hosting platform
- ✅ Automated CI/CD pipeline
- ✅ Monitoring and alerting
- ✅ Secure remote access
- ✅ Persistent storage on NAS
- ✅ Room to grow and experiment

**Total setup time:** 2-3 days for core functionality

**Maintenance:** ~2 hours/week for updates and monitoring