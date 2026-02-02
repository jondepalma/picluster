# Claude Code Development Context

**Project Environment:** Raspberry Pi K3s Cluster
**Last Updated:** 2025-11-28
**Status:** Current and verified against live cluster

---

## Infrastructure Overview

### Cluster Nodes

| Node | IP | Role | RAM | Storage | Current Usage |
|------|-----|------|-----|---------|---------------|
| **pi-control** | 10.0.0.10 | Control plane | 8GB | 1TB NVMe + 256GB SD | 1% CPU, 17% RAM |
| **pi-worker-1** | 10.0.0.11 | Worker + Dev Node | 8GB | 256GB SD | 1% CPU, 21% RAM |
| **pi-worker-2** | 10.0.0.12 | Worker | 8GB | 256GB SD | 1% CPU, 11% RAM |

**Key Points:**
- **pi-worker-1** is the **development workstation** - SSH here for kubectl and development
- **pi-control** has NVMe at `/mnt/nvme` for high-performance storage
- All nodes are Raspberry Pi 5 with ARM64 architecture

### External Infrastructure
- **QNAP NAS:** 10.0.0.5 (NFS storage server)
- **Network:** 10.0.0.0/24
- **Static IPs:** 10.0.0.1-10.0.0.50
- **Dynamic IPs:** 10.0.0.51-10.0.0.254
- **MetalLB Pool:** 10.0.0.30-10.0.0.50

---

## Kubernetes Configuration

### Software Versions

| Component | Version | Notes |
|-----------|---------|-------|
| **K3s** | v1.30.x | Need to run: `kubectl version` |
| **MetalLB** | v0.15.2 | Load balancer |
| **Traefik** | K3s default | Ingress controller |
| **NFS Provisioner** | 4.0.18 | Helm chart |
| **Traefik** | K3s bundled | **v3.6.0 (Helm)** | Custom Helm installation in traefik namespace |
| **MySQL** | mysql:8.0 | **mysql:8.0** | ✅ Confirmed |

### Namespaces

| Namespace | Purpose | Key Services |
|-----------|---------|--------------|
| `dev` | Development | PostgreSQL, Gitea (+ Registry), Valkey |
| `prod` | Production | MySQL, Cloudflare Tunnel |
| `ghost` | Blog platform | Ghost CMS |
| `harbor` | Container registry | ⚠️ Harbor (failed - ARM64 issues) |
| `monitoring` | Observability | Beszel (or planned) |
| `tailscale` | VPN | Subnet router |
| `kube-system` | System | K3s, NFS |
| `metallb-system` | Networking | MetalLB |
| `traefik` | Networking | Ingress |

### Storage Classes

| Class | Provisioner | Location | Speed | Use Case |
|-------|-------------|----------|-------|----------|
| `local-path` | rancher.io/local-path | SD cards | ~90 MB/s | Logs, temp files, cache |
| `local-path-nvme` | rancher.io/local-path | pi-control NVMe | ~3000 MB/s | **Dev databases** |
| `nfs-client` | nfs-subdir-external-provisioner | QNAP NAS | ~110 MB/s | Bulk storage, files |
| `nfs-db` | nfs-subdir-external-provisioner | QNAP NAS | ~110 MB/s | **Prod databases** |

**Storage Strategy:**
- Development databases → `local-path-nvme` (speed)
- Production databases → `nfs-db` (high availability)
- Application files/repos → `nfs-client` (shared, persistent)
- Temporary data → `local-path` (disposable)

### Control Plane Scheduling

**Important:** pi-control has a taint to prevent normal workloads:
```yaml
taints:
- key: CriticalAddonsOnly
  value: "true"
  effect: NoExecute
```

**To schedule on pi-control** (for dev databases using NVMe):
```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: pi-control
  tolerations:
  - key: CriticalAddonsOnly
    operator: Equal
    value: "true"
    effect: NoExecute
```

---

## Deployed Services

### Development Environment (Namespace: `dev`)

#### PostgreSQL Development Database
**Service Name:** `postgresql-dev.dev.svc.cluster.local`
**Pod Name:** `postgresql-dev-0` (StatefulSet)
**Image:** `bitnami/postgresql:latest`
**Storage:** 30Gi on `local-path-nvme` (pi-control NVMe)
**Resources:** 6m CPU, 66Mi RAM (actual usage)
**Shared by:** Gitea (and available for other dev apps)
**Databases:** `gitea`, `devdb` (Note: `harbor` db exists but Harbor deployment failed)
**Secret:** `postgresql-dev` (keys: `password`, `postgres-password`, `replication-password`)

**Connection (from inside cluster):**
```
Host: postgresql-dev.dev.svc.cluster.local
Port: 5432
Database: devdb
Username: postgres
Password: kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 -d
```

**Connection String:**
```
postgresql://postgres:<password>@postgresql-dev.dev.svc.cluster.local:5432/devdb
```

**Run psql commands via kubectl exec (no local psql needed):**
```bash
# Interactive psql session
kubectl exec -it -n dev postgresql-dev-0 -- psql -U postgres -d devdb

# Run a single command (e.g., create database)
kubectl exec -it -n dev postgresql-dev-0 -- psql -U postgres -c "CREATE DATABASE mydb;"

# List databases
kubectl exec -it -n dev postgresql-dev-0 -- psql -U postgres -c "\l"
```

**Port-forward for bare-metal dev:**
```bash
kubectl port-forward -n dev svc/postgresql-dev 5432:5432
# Then connect to localhost:5432
```

**MetalLB LoadBalancer Access:**
```bash
# Direct access from LAN or Tailscale
psql -h 10.0.0.32 -U postgres -d devdb

# Or use hostname (after adding to Windows hosts file)
psql -h pi-cluster-postgresql-dev -U postgres -d devdb
```

#### Redis Stack Development
**Service Name:** `redis-stack-dev.dev.svc.cluster.local`
**Pod Name:** `redis-stack-dev-*` (Deployment)
**Image:** `redis/redis-stack:latest`
**Storage:** 10Gi on `local-path-nvme` (pi-control NVMe)
**Resources:** 250m CPU, 512Mi RAM (requests)
**Purpose:** Development caching, RedisJSON storage, experimentation

**Modules Loaded:**
- RedisJSON (v20809) - JSON document storage
- RedisSearch (v2.10.20) - Full-text search and secondary indexing
- RedisTimeSeries (v11206) - Time-series data storage
- RedisBloom (v2.8.16) - Probabilistic data structures
- RedisGears (v2.0.20) - Serverless engine with JavaScript support

**Connection (from inside cluster):**
```
Host: redis-stack-dev.dev.svc.cluster.local
Port: 6379 (Redis)
Port: 8001 (Redis Insight UI)
Password: kubectl get secret redis-dev-credentials -n dev -o jsonpath='{.data.redis-password}' | base64 -d
```

**Connection String:**
```
redis://:password@redis-stack-dev.dev.svc.cluster.local:6379
```

**Run redis-cli commands via kubectl exec:**
```bash
# Interactive redis-cli session
kubectl exec -it -n dev deployment/redis-stack-dev -- sh -c 'redis-cli -a "$REDIS_PASSWORD"'

# Run a single command
kubectl exec -n dev deployment/redis-stack-dev -- sh -c 'redis-cli -a "$REDIS_PASSWORD" SET key value'

# Test RedisJSON
kubectl exec -n dev deployment/redis-stack-dev -- sh -c 'redis-cli -a "$REDIS_PASSWORD" JSON.SET poll:1 $ "{\"question\":\"Test?\",\"votes\":5}"'
```

**MetalLB LoadBalancer Access:**
```bash
# Direct Redis access from LAN or Tailscale
redis-cli -h 10.0.0.33 -p 6379 -a PASSWORD

# Or use hostname
redis-cli -h pi-cluster-redis-dev -p 6379 -a PASSWORD

# Redis Insight UI
http://10.0.0.33:8001
http://pi-cluster-redis-dev:8001
```

**Database Allocation (DEV):**
- DB 0-15: Available for experiments (Harbor deployment failed)

**Configuration:** `applications/redis/redis-stack-dev.yaml`
**LoadBalancer:** `networking/metallb/redis-stack-dev-loadbalancer.yaml`
**Documentation:** `applications/redis/REDIS-DEPLOYMENT-GUIDE.md`

#### MongoDB Development Database
**Service Name:** `mongodb-dev.dev.svc.cluster.local`
**Image:** `mongo:7.0`
**Storage:** 30Gi on `local-path-nvme` (pi-control NVMe)
**Resources:** 100m CPU, 256Mi RAM
**Deployment:** StatefulSet (mongodb-dev-0)
**Credentials:** Stored in `mongodb-dev-credentials` secret

**Connection (from inside cluster):**
```bash
# Via ClusterIP
mongodb://mongodb-dev.dev.svc.cluster.local:27017

# Via MetalLB LoadBalancer (from anywhere on network or Tailscale)
mongodb://10.0.0.34:27017

# Windows hosts file entry
10.0.0.34    pi-cluster-mongodb-dev
mongodb://pi-cluster-mongodb-dev:27017
```

**Get credentials:**
```bash
kubectl get secret mongodb-dev-credentials -n dev -o jsonpath='{.data.mongodb-root-username}' | base64 -d
kubectl get secret mongodb-dev-credentials -n dev -o jsonpath='{.data.mongodb-root-password}' | base64 -d
kubectl get secret mongodb-dev-credentials -n dev -o jsonpath='{.data.mongodb-database}' | base64 -d
```

**Access MongoDB shell:**
```bash
# Via kubectl exec
kubectl exec -it mongodb-dev-0 -n dev -- mongosh \
  -u $(kubectl get secret mongodb-dev-credentials -n dev -o jsonpath='{.data.mongodb-root-username}' | base64 -d) \
  -p $(kubectl get secret mongodb-dev-credentials -n dev -o jsonpath='{.data.mongodb-root-password}' | base64 -d) \
  --authenticationDatabase admin

# Via port-forward (localhost:27017)
kubectl port-forward -n dev mongodb-dev-0 27017:27017

# Via MetalLB LoadBalancer (no port-forward needed)
mongosh mongodb://10.0.0.34:27017 -u <username> -p <password> --authenticationDatabase admin
```

**Automated Backups:**
- Schedule: Daily at 3:00 AM
- Retention: 14 days
- Storage: NFS `/share/kubernetes-backups/mongodb-dev/`
- Method: mongodump with gzip compression
- CronJob: `mongodb-dev-backup` in `dev` namespace

**Configuration:** `applications/mongodb/mongodb-dev.yaml`
**Backup CronJob:** `applications/mongodb/mongodb-dev-backup-cronjob.yaml`
**LoadBalancer:** `networking/metallb/mongodb-dev-loadbalancer.yaml`

#### Neo4j Development Database
**Service Name:** `neo4j-dev.dev.svc.cluster.local`
**Image:** `neo4j:5.15-community`
**Storage:** 50Gi on `local-path-nvme` (pi-control NVMe)
**Resources:** 500m CPU, 2Gi RAM
**Deployment:** StatefulSet (neo4j-dev-0)
**Credentials:** Stored in `neo4j-dev-credentials` secret

**Connection (from inside cluster):**
```bash
# Browser UI (HTTP)
http://neo4j-dev.dev.svc.cluster.local:7474

# Bolt Protocol
bolt://neo4j-dev.dev.svc.cluster.local:7687
```

**Connection (via MetalLB LoadBalancer):**
```bash
# Browser UI - from anywhere on network or Tailscale
http://10.0.0.35:7474

# Bolt Protocol
bolt://10.0.0.35:7687

# Windows hosts file entry
10.0.0.35    pi-cluster-neo4j-dev
http://pi-cluster-neo4j-dev:7474
bolt://pi-cluster-neo4j-dev:7687
```

**Get credentials:**
```bash
# NEO4J_AUTH format (neo4j/<password>)
kubectl get secret neo4j-dev-credentials -n dev -o jsonpath='{.data.NEO4J_AUTH}' | base64 -d

# Password only
kubectl get secret neo4j-dev-credentials -n dev -o jsonpath='{.data.neo4j-password}' | base64 -d
```

**Access Neo4j shell:**
```bash
# Via kubectl exec
kubectl exec -it neo4j-dev-0 -n dev -- cypher-shell \
  -u neo4j \
  -p $(kubectl get secret neo4j-dev-credentials -n dev -o jsonpath='{.data.neo4j-password}' | base64 -d)

# Via port-forward for Browser UI (localhost:7474)
kubectl port-forward -n dev neo4j-dev-0 7474:7474 7687:7687

# Via MetalLB LoadBalancer (no port-forward needed)
# Open browser: http://10.0.0.35:7474
# Connect URL: bolt://10.0.0.35:7687
# Username: neo4j
# Password: <from secret>
```

**Memory Configuration:**
- Heap: 1G initial, 2G max
- Page cache: 1G
- Total limit: 4Gi

**Automated Backups:**
- Schedule: Daily at 3:30 AM
- Retention: 14 days
- Storage: NFS `/share/kubernetes-backups/neo4j-dev/`
- Method: Metadata backup (Community Edition limitation)
- CronJob: `neo4j-dev-backup` in `dev` namespace
- Note: Full backup requires Enterprise Edition

**Configuration:** `applications/neo4j/neo4j-dev.yaml`
**Backup CronJob:** `applications/neo4j/neo4j-dev-backup-cronjob.yaml`
**LoadBalancer:** `networking/metallb/neo4j-dev-loadbalancer.yaml`

#### Gitea Git Server
**Service Name:** `gitea-http.dev.svc.cluster.local`  
**Image:** `docker.gitea.com/gitea:1.24.6-rootless`  
**Storage:** 50Gi on `nfs-client` (QNAP NAS)  
**Resources:** 6m CPU, 108Mi RAM  
**Database:** Uses PostgreSQL Dev  
**External Access:** Via Cloudflare Tunnel (git.yourdomain.com)

**Includes Valkey Cluster:**
- 3-node cluster: gitea-valkey-cluster-0/1/2
- Version: 8.1.3-debian-12
- Storage: 8Gi each on `local-path` (SD cards)
- Purpose: Gitea session/cache management
- Resources: ~35m CPU, 22Mi RAM total

**Connection (from inside cluster):**
```
Host: gitea-http.dev.svc.cluster.local
Port: 3000
```

**Admin access:**
```
Username: (set during installation)
URL: https://git.yourdomain.com
```

#### Harbor Container Registry

**❌ Harbor Deployment Abandoned** (2025-11-28)

Harbor is **NOT compatible with ARM64 architecture** (Raspberry Pi). Official Harbor images are AMD64-only and fail with `exec format error`.

**See:** `/applications/harbor/ARM64-COMPATIBILITY-ISSUE.md` for full details.

**Service Name:** `harbor-portal.harbor.svc.cluster.local`
**Image:** Various Harbor components via Helm chart
**Namespace:** `harbor`
**Storage:** 100Gi (registry) + 10Gi (Trivy) on `nfs-client` (QNAP NAS)
**Database:** Uses PostgreSQL Dev (`harbor` database)
**Cache:** Uses Redis Stack Dev (DB 0, 1, 2, 5)
**External Access:** Via Cloudflare Tunnel (registry.jondepalma.net)

**ABANDONED: Architecture:**
- Uses existing `postgresql-dev` database (harbor/harbor user)
- Uses existing `redis-stack-dev` (Harbor requires DB 0 for core - conflicts with redis-cache-prod/Ghost)
- No internal PostgreSQL or Redis instances
- Trivy vulnerability scanner enabled
- Aligned with dev infrastructure (postgresql-dev + redis-stack-dev)

**ABANDONED: Connection (from inside cluster):**
```
Registry: registry.jondepalma.net
Web UI: https://harbor-portal.harbor.svc.cluster.local:443
Admin Username: admin
Admin Password: kubectl get secret harbor-admin-password -n harbor -o jsonpath='{.data.password}' | base64 -d
```

**ABANDONED: Docker login:**
```bash
docker login registry.jondepalma.net
# Username: admin
# Password: (from harbor-admin-password secret)

# Push images
docker tag myimage:latest registry.jondepalma.net/project/myimage:latest
docker push registry.jondepalma.net/project/myimage:latest
```

**ABANDONED: Database Details:**
```
Database: harbor (on postgresql-dev.dev.svc.cluster.local:5432)
User: harbor
Password: kubectl get secret harbor-database -n harbor -o jsonpath='{.data.password}' | base64 -d
```

**ABANDONED: Redis Database Allocation (on redis-stack-dev):**
- DB 0: Harbor core (required by Harbor library)
- DB 1: Harbor job service
- DB 2: Harbor registry
- DB 5: Harbor Trivy adapter

**ABANDONED: Cross-Namespace Secrets:**
- `harbor-redis` in `harbor` namespace (key: `REDIS_PASSWORD`) - sourced from redis-dev-credentials
- `harbor-database` in `harbor` namespace - created independently
- `harbor-database` in `dev` namespace - copy for init job only

**ABANDONED: Resources:** ~900m CPU, ~1.2Gi RAM (saves ~500Mi vs internal DB/Redis)

**ABANDONED: Configuration:** `applications/harbor/harbor-values.yaml`
**ABANDONED: Secret Templates:** `secrets/secret-templates/harbor-*.yaml.tmpl`

**⚠️ NOTE:** Harbor deployment FAILED due to incomplete ARM64 support. See Gitea Container Registry below for working alternative.

#### Gitea Container Registry (Active Alternative to Harbor)
**Service Name:** `git.jondepalma.net`
**Component:** Built into Gitea (already deployed)
**Namespace:** `dev`
**Storage:** Uses existing Gitea storage (nfs-client)
**Database:** Uses existing Gitea database on PostgreSQL Dev
**External Access:** Via Cloudflare Tunnel (git.jondepalma.net)
**Status:** ✅ Operational (deployed 2025-11-28)

**Why Gitea Instead of Harbor:**
- Harbor jobservice does not work reliably on ARM64
- Trivy scanner has ARM64 compatibility problems
- Harbor requires ~1.1 CPU, ~1.4Gi RAM, 125Gi storage
- Gitea Registry is ARM64-native, zero additional resources, already installed

**Architecture:**
- Built-in to Gitea 1.24.6 (enabled by default)
- Zero additional resources required
- Integrated authentication with Gitea users
- Full ARM64 support
- Uses Gitea's existing PostgreSQL database and storage

**Docker Registry Access:**
```bash
# Login
docker login git.jondepalma.net
# Username: [gitea-username]
# Password: [gitea-access-token with write:package scope]

# Push images (note: username is required in path)
docker tag myimage:latest git.jondepalma.net/[username]/myimage:latest
docker push git.jondepalma.net/[username]/myimage:latest

# Pull images
docker pull git.jondepalma.net/[username]/myimage:latest
```

**Kubernetes Image Pull Secrets:**
```bash
# Dev namespace
kubectl get secret gitea-registry -n dev

# Prod namespace
kubectl get secret gitea-registry -n prod

# Use in deployments
spec:
  imagePullSecrets:
  - name: gitea-registry
```

**Gitea Actions CI/CD:**
- **Runner:** `pi-worker-1-runner` (act_runner v0.2.11)
- **Location:** `/home/pi/.runner` on pi-worker-1
- **Service:** `gitea-runner.service` (systemd)
- **Status:** Check with `sudo systemctl status gitea-runner`
- **Logs:** `sudo journalctl -u gitea-runner -f`
- **Admin UI:** https://git.jondepalma.net/admin/actions/runners

**Vulnerability Scanning:**
- **Trivy:** v0.67.2 (installed on pi-worker-1 at `/usr/local/bin/trivy`)
- **Purpose:** Container image vulnerability scanning
- **Usage:** `trivy image git.jondepalma.net/[username]/[app]:[tag]`
- **Workflow Integration:** Available for Gitea Actions workflows
- **Documentation:** `docs/gitea-workflows/ARM64-REGISTRY-EVALUATION.md`

**Resources:** ~20m CPU, ~50Mi RAM additional (minimal overhead)

**Documentation:**
- Setup Guide: `docs/gitea-workflows/GITEA-REGISTRY-SETUP-GUIDE.md`
- Workflows: `docs/gitea-workflows/`
- Evaluation: `docs/gitea-workflows/ARM64-REGISTRY-EVALUATION.md`

### Production Environment

#### MySQL Database (Namespace: `prod`)
**Service Name:** `mysql-ghost.prod.svc.cluster.local`
**Image:** `mysql:8.0`
**Storage:** 20Gi on `nfs-db` (QNAP NAS)
**Resources:** 5m CPU, 402Mi RAM
**Purpose:** Ghost blog database

**Connection (from inside cluster):**
```
Host: mysql-ghost.prod.svc.cluster.local
Port: 3306
Database: ghost
Username: root
Password: kubectl get secret mysql-ghost -n prod -o jsonpath='{.data.root-password}' | base64 -d
```

#### Redis Cache Production (Namespace: `prod`)
**Service Name:** `redis-cache-prod.prod.svc.cluster.local` (alias: `redis-master.prod.svc.cluster.local`)
**Pod Name:** `redis-cache-prod-*` (Deployment)
**Image:** `redis:7-alpine`
**Storage:** 10Gi on `local-path-nvme` (pi-control NVMe)
**Resources:** 100m CPU, 256Mi RAM (requests)
**Purpose:** Ghost blog caching, API response caching

**Connection (from inside cluster):**
```
Host: redis-master.prod.svc.cluster.local (or redis-cache-prod.prod.svc.cluster.local)
Port: 6379
Password: kubectl get secret redis-prod-credentials -n prod -o jsonpath='{.data.redis-password}' | base64 -d
```

**Run redis-cli commands via kubectl exec:**
```bash
# Interactive redis-cli session
kubectl exec -it -n prod deployment/redis-cache-prod -- sh -c 'redis-cli -a "$REDIS_PASSWORD"'

# Run a single command
kubectl exec -n prod deployment/redis-cache-prod -- sh -c 'redis-cli -a "$REDIS_PASSWORD" INFO stats'
```

**Database Allocation (PROD):**
- DB 0: Ghost blog caching (page fragments, queries)
- DB 1: Session storage (user sessions, login tokens)
- DB 2-15: Reserved for future API caching and services

**Configuration:** `applications/redis/redis-cache-prod.yaml`
**Documentation:** `applications/redis/REDIS-DEPLOYMENT-GUIDE.md`

**IMPORTANT - Cross-Namespace Secret Access:**
- The `redis-prod-credentials` secret exists in the `prod` namespace
- Ghost Blog (in `ghost` namespace) also needs this secret to access Redis
- Kubernetes does NOT support cross-namespace secret references
- Secret must be duplicated in both `prod` and `ghost` namespaces
- When rotating credentials, update BOTH namespaces
- Automated rotation script in `secrets/secrets-strategy.md` handles both

#### Ghost Blog (Namespace: `ghost`)
**Service Name:** `ghost.ghost.svc.cluster.local`
**Image:** `ghost:5.130.5`
**Storage:** 10Gi on `nfs-client` (content/uploads)
**Database:** `mysql-ghost.prod.svc.cluster.local` (MySQL in prod namespace)
**Cache:** `redis-cache-prod.prod.svc.cluster.local` (Redis DB 0 in prod namespace)
**External Access:** Via Cloudflare Tunnel (https://blog.jondepalma.net)

**Redis Cache Configuration:**
- Ghost uses Redis for caching (page fragments, queries, API responses)
- Configured via environment variables in deployment manifest
- Cache adapter: `redis` (enabled via `cache__adapter` env var)
- Connection: `redis-cache-prod.prod.svc.cluster.local:6379` (DB 0)
- Authentication: `redis-prod-credentials` secret (duplicated in `ghost` namespace)

**Configuration:** `applications/ghost-blog/05-ghost-deployment.yaml`

### Monitoring

#### Beszel Monitoring Stack (Namespace: `monitoring`)
**Service Name:** `beszel-hub.monitoring.svc.cluster.local`
**Hub Image:** `henrygd/beszel:latest`
**Agent Image:** `henrygd/beszel-agent:latest`
**Storage:** 5Gi on `local-path-nvme` (pi-control NVMe)
**Resources (Hub):** 100m CPU request, 128Mi RAM request
**Resources (Agent):** 50m CPU request, 32Mi RAM request per node

**Architecture:**
- **Hub:** Central monitoring server on pi-control (NVMe storage)
- **Agents:** DaemonSet running on all nodes with hostNetwork
- **Communication:** SSH-based (hub initiates connections to agents on node IPs)
- **Port:** 45876 (agent SSH port)

**Access:**
- Web UI: https://monitor.jondepalma.net (via Cloudflare Tunnel)
- Internal: http://beszel-hub.monitoring.svc.cluster.local:8090

**Important Configuration Notes:**
- Hub requires `CriticalAddonsOnly` toleration to schedule on pi-control
- Agents use hostNetwork: true for node-level monitoring
- Containerd socket path: `/run/k3s/containerd/containerd.sock` (K3s specific)
- Mount path: `/beszel_data` (note underscore, not slash)
- SSH keys stored in Kubernetes secret: `beszel-agent-key`

**NetworkPolicy Considerations:**
- Must allow egress to node IPs (10.0.0.0/24) on port 45876 for SSH connections
- Must allow ingress from Cloudflare namespace using label: `kubernetes.io/metadata.name: cloudflare`
- Default namespace labels: `kubernetes.io/metadata.name=<namespace-name>` (not `name=<namespace-name>`)

**Configuration:** `applications/beszel/beszel-deployment.yaml`

### Database Backups

#### Automated Backup Strategy
**Storage Location:** `/share/kubernetes-backups/` on QNAP NAS (10.0.0.5)
**Method:** Direct NFS mounts (not PVCs) for consistency with secrets backup location
**Format:** Compressed SQL dumps (.sql.gz)

#### PostgreSQL Backup (Namespace: `dev`)
**CronJob:** `postgresql-dev-backup`
**Schedule:** Daily at 2:00 AM EST
**Retention:** 14 days
**Method:** `pg_dumpall` (backs up ALL databases + roles)
**Storage:** `/share/kubernetes-backups/databases/postgresql-dev/`

**Databases Backed Up:**
- `gitea` - Gitea database
- `devdb` - Development database
- `postgres` - System database
- Plus all roles (excluding passwords via `--no-role-passwords`)

**Image:** `bitnami/postgresql:latest`
**Resources:** 256Mi RAM request, 512Mi limit

**Important Notes:**
- Uses `pg_dumpall` instead of `pg_dump` to backup all databases in one file
- Runs on pi-control (requires `CriticalAddonsOnly` toleration)
- Password from secret: `postgresql-dev` key `postgres-password`

#### MySQL Backup (Namespace: `prod`)
**CronJob:** `mysql-ghost-backup`
**Schedule:** Daily at 2:30 AM EST (staggered 30 minutes from PostgreSQL)
**Retention:** 30 days (longer for production data)
**Method:** `mysqldump` with `--single-transaction` for consistency
**Storage:** `/share/kubernetes-backups/databases/mysql-ghost/`

**Database Backed Up:**
- `ghost` - Ghost blog database (includes routines, triggers, events)

**Image:** `mysql:8.0`
**Resources:** 256Mi RAM request, 512Mi limit

**Important Notes:**
- Uses `ghost` user (not `root`) - root user restricted to localhost only
- Password from secret: `mysql-ghost` key `password`
- Includes `--routines --triggers --events` for complete backup

**Common Backup Operations:**
```bash
# Manually trigger PostgreSQL backup
kubectl create job --from=cronjob/postgresql-dev-backup manual-pg-backup-$(date +%Y%m%d) -n dev

# Manually trigger MySQL backup
kubectl create job --from=cronjob/mysql-ghost-backup manual-mysql-backup-$(date +%Y%m%d) -n prod

# View backup job logs
kubectl logs -n dev -l job-name=<job-name>

# List backups on NAS
ls -lh /share/kubernetes-backups/databases/postgresql-dev/
ls -lh /share/kubernetes-backups/databases/mysql-ghost/
```

**Restore Process:**
```bash
# PostgreSQL restore (all databases)
kubectl exec -i postgresql-dev-0 -n dev -- psql -U postgres < backup-file.sql

# MySQL restore
kubectl exec -i mysql-ghost-0 -n prod -- mysql -u ghost -p ghost < backup-file.sql
```

**Configuration:**
- PostgreSQL: `applications/postgres/postgresql-backup-cronjob.yaml`
- MySQL: `applications/ghost-blog/mysql-backup-cronjob.yaml`

### Infrastructure Services

#### Cloudflare Tunnel (Namespace: `prod`)
**Purpose:** External HTTPS access to cluster services  
**Image:** (need to check: `kubectl get deployment -n cloudflare cloudflared -o jsonpath='{.spec.template.spec.containers[0].image}'`)  
**Public Endpoints:**
- blog.yourdomain.com → Ghost
- git.yourdomain.com → Gitea
- (add others as configured)

**Configuration:** `networking/cloudflare/cloudflared-deployment.yaml`

#### Tailscale VPN (Namespace: `tailscale`)
**Purpose:** Secure remote access to cluster network  
**Mode:** Subnet router (advertises 10.0.0.0/24)  
**Image:** (need to check: `kubectl get deployment -n tailscale -l app=tailscale-subnet-router -o jsonpath='{.spec.template.spec.containers[0].image}'`)  
**Device Name:** pi-cluster-k8s (in Tailscale admin)

**Access cluster via Tailscale:**
```bash
# From any device with Tailscale installed
ssh pi@10.0.0.10  # pi-control
ssh pi@10.0.0.11  # pi-worker-1
kubectl get nodes  # If kubeconfig is set up
```

**Configuration:** `networking/tailscale/tailscale-deployment.yaml`

---

#### Traefik Ingress Controller (Namespace: `traefik`)
**Deployment:** Custom Helm installation (not K3s bundled)  
**Image:** `traefik:v3.6.0`  
**Helm Chart:** traefik-37.3.0  
**Service:** LoadBalancer (MetalLB assigns IP from pool)

**Configuration:**
- Dashboard enabled (port 8080)
- Prometheus metrics enabled (port 9100)
- Entry points: web (80), websecure (443)
- TLS enabled on websecure
- Kubernetes CRD and Ingress providers enabled

**Access Dashboard (dev only):**
```bash
kubectl port-forward -n traefik deployment/traefik 8080:8080
# Then visit: http://localhost:8080/dashboard/
```

**Manifest:** `networking/traefik/traefik-deployment.yaml`

---

## Development Workflow

### Development Environment on pi-worker-1

**Location:** `/home/pi/dev-projects/` (or your preferred path)  
**Tools Installed:**
- Python 3.11+
- Node.js 20+
- Docker
- kubectl (configured for cluster access)
- Git

**VS Code Access:**
```bash
# SSH config entry (on your Windows machine):
Host pi-dev
    HostName 10.0.0.11
    User pi
    ForwardAgent yes
```

Then connect via VS Code Remote-SSH to `pi-dev`

### Accessing Cluster Databases from Bare-Metal

**Method 1: Port-Forwarding (Development)**
```bash
# On pi-worker-1
kubectl port-forward -n dev svc/postgresql-dev 5432:5432 &

# Now your app can connect to localhost:5432
```

**Method 2: Deploy to Cluster (Testing)**
```bash
# Build image (ARM64)
docker buildx build --platform linux/arm64 -t your-app:v1 .

# Push to registry
docker push your-username/your-app:v1

# Deploy
kubectl apply -f deployment.yaml
```

### Environment Variables Pattern

**Local Development (.env file):**
```bash
DATABASE_URL=postgresql://postgres:password@localhost:5432/devdb
GITEA_URL=http://localhost:3000
ENVIRONMENT=development
```

**Kubernetes Deployment:**
```yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: database-url
  - name: GITEA_URL
    value: "http://gitea-http.dev.svc.cluster.local:3000"
  - name: ENVIRONMENT
    value: "production"
```

---

## Resource Availability

### Current Usage (Very Low!)

| Resource | Used | Total | Available | Percentage |
|----------|------|-------|-----------|------------|
| **CPU** | 184m | 12 cores | 11.8 cores | 98.5% free |
| **RAM** | 4055Mi | 24Gi | 20Gi | 83% free |
| **NVMe** | 30Gi | 1TB | 970Gi | 97% free |

**Implication:** Massive headroom for MongoDB, Redis, and additional applications!

### Storage Allocation

| Type | Used | Total | Available | Purpose |
|------|------|-------|-----------|---------|
| **NVMe (pi-control)** | 30Gi | 1TB | 970Gi | Dev databases |
| **NFS (QNAP)** | ~100Gi | ~500Gi | ~400Gi | Prod databases, files |
| **SD Cards** | Minimal | 768Gi | ~750Gi | OS, logs, cache |

---

## Common Commands

### Cluster Management
```bash
# Get cluster status
kubectl get nodes
kubectl top nodes

# Check all running pods
kubectl get pods -A

# View pod logs
kubectl logs -f <pod-name> -n <namespace>

# Execute into pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# Port-forward service
kubectl port-forward -n <namespace> svc/<service-name> <local-port>:<remote-port>
```

### Database Access

**PostgreSQL Dev:**
```bash
# Get password
kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 -d

# Connect from within cluster
kubectl exec -it postgresql-dev-0 -n dev -- psql -U postgres -d devdb

# Port-forward for local access
kubectl port-forward -n dev svc/postgresql-dev 5432:5432
psql -h localhost -U postgres -d devdb
```

**MySQL (Ghost):**
```bash
# Get password
kubectl get secret mysql-ghost -n prod -o jsonpath='{.data.root-password}' | base64 -d

# Connect from within cluster
kubectl exec -it mysql-ghost-0 -n prod -- mysql -u root -p

# Port-forward for local access
kubectl port-forward -n prod svc/mysql-ghost 3306:3306
mysql -h localhost -u root -p ghost
```

**Redis Stack (DEV):**
```bash
# Get password
kubectl get secret redis-dev-credentials -n dev -o jsonpath='{.data.redis-password}' | base64 -d

# Connect from within cluster
kubectl exec -it -n dev deployment/redis-stack-dev -- sh -c 'redis-cli -a "$REDIS_PASSWORD"'

# Via MetalLB LoadBalancer (from LAN or Tailscale)
redis-cli -h 10.0.0.33 -p 6379 -a PASSWORD
redis-cli -h pi-cluster-redis-dev -p 6379 -a PASSWORD

# Redis Insight UI
http://10.0.0.33:8001
http://pi-cluster-redis-dev:8001
```

**Redis Cache (PROD):**
```bash
# Get password
kubectl get secret redis-prod-credentials -n prod -o jsonpath='{.data.redis-password}' | base64 -d

# Connect from within cluster
kubectl exec -it -n prod deployment/redis-cache-prod -- sh -c 'redis-cli -a "$REDIS_PASSWORD"'
```

### Storage Management
```bash
# View storage classes
kubectl get sc

# View persistent volumes
kubectl get pv

# View persistent volume claims
kubectl get pvc -A

# Check storage usage by PVC
kubectl get pvc -A -o custom-columns=\
NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
STORAGECLASS:.spec.storageClassName,\
CAPACITY:.spec.resources.requests.storage,\
STATUS:.status.phase
```

### Secrets Management
```bash
# List secrets in namespace
kubectl get secrets -n <namespace>

# Get secret value
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.<key>}' | base64 -d

# Create secret from literal
kubectl create secret generic my-secret \
  --from-literal=key=value \
  -n <namespace>

# Create secret from file
kubectl create secret generic my-secret \
  --from-file=credentials.json \
  -n <namespace>
```

---

## Network Access Patterns

### Internal Service Communication
```
<service-name>.<namespace>.svc.cluster.local:<port>

Examples:
postgresql-dev.dev.svc.cluster.local:5432
gitea-http.dev.svc.cluster.local:3000
mysql-ghost.prod.svc.cluster.local:3306
```

### MetalLB LoadBalancer Services

Static IP access to cluster services on LAN (10.0.0.0/24):

| Service | IP Address | Ports | Hostname | Purpose |
|---------|------------|-------|----------|---------|
| **Traefik** | 10.0.0.30 | 80, 443 | - | HTTP/HTTPS ingress |
| **Gitea SSH** | 10.0.0.31 | 22 | - | Git over SSH |
| **PostgreSQL DEV** | 10.0.0.32 | 5432 | pi-cluster-postgresql-dev | Database access |
| **Redis Stack DEV** | 10.0.0.33 | 6379, 8001 | pi-cluster-redis-dev | Redis + Insight UI |

**Windows Hosts File Configuration:**
```
# Add to C:\Windows\System32\drivers\etc\hosts
10.0.0.32    pi-cluster-postgresql-dev
10.0.0.33    pi-cluster-redis-dev
```

**Access from LAN or Tailscale:**
```bash
# PostgreSQL
psql -h pi-cluster-postgresql-dev -U postgres -d devdb

# Redis
redis-cli -h pi-cluster-redis-dev -p 6379 -a PASSWORD

# Redis Insight UI
http://pi-cluster-redis-dev:8001
```

See: `networking/metallb/README-LOADBALANCERS.md`

### External Access Methods

| Method | Use Case | Example |
|--------|----------|---------|
| **Cloudflare Tunnel** | Public HTTPS access | https://blog.yourdomain.com |
| **Tailscale VPN** | Secure remote access | ssh pi@10.0.0.11 |
| **MetalLB LoadBalancer** | LAN + Tailscale access | postgresql://10.0.0.32:5432 |
| **kubectl port-forward** | Development/debugging | localhost:5432 |

---

## Best Practices for This Cluster

### Docker Images
- **Always build for ARM64:** `--platform linux/arm64`
- Use Alpine or slim base images to reduce size
- Tag with version numbers, not just `latest`
- Test locally on Pi before deploying

### Resource Management
- Always set resource requests and limits
- Monitor usage with `kubectl top`
- Use appropriate storage classes (NVMe for dev DBs, NFS for prod)
- Keep production databases on NFS for HA

### Database Connections
- Use connection pooling in applications
- Implement retry logic for cluster connectivity
- Close connections properly
- Use Kubernetes secrets for credentials
- Never hardcode passwords in manifests

### Working with Kubernetes Secrets
**Always retrieve passwords from secrets - never hardcode them:**
```bash
# Get password and use in a command
export PGPASSWORD=$(kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 -d)

# Or for scripts, inject directly:
PG_PASS=$(kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 -d)
```

**For .env files in local development:**
```bash
# Auto-populate .env from secret
PG_PASS=$(kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 -d)
sed -i "s|YOUR_PASSWORD_HERE|${PG_PASS}|" .env
```

### Working with StatefulSets (Databases)
Databases run as StatefulSets, not Deployments. Pod names follow the pattern `<statefulset-name>-<ordinal>`:

| StatefulSet | Pod Name | Namespace |
|-------------|----------|-----------|
| postgresql-dev | `postgresql-dev-0` | dev |
| mysql-ghost | `mysql-ghost-0` | prod |
| gitea-valkey-cluster | `gitea-valkey-cluster-0/1/2` | dev |

**Use kubectl exec with the pod name (not service or deployment):**
```bash
# Correct - use pod name
kubectl exec -it -n dev postgresql-dev-0 -- psql -U postgres

# Wrong - services/deployments don't work with exec
kubectl exec -it -n dev svc/postgresql-dev -- psql -U postgres  # ERROR
kubectl exec -it -n dev deploy/postgresql-dev -- psql -U postgres  # ERROR
```

### Storage Strategy
- **Dev databases** → local-path-nvme (speed for development)
- **Prod databases** → nfs-db (HA and backups)
- **Application files** → nfs-client (shared, persistent)
- **Temporary/cache** → local-path (fast, disposable)

### Security
- Use secrets for all sensitive data
- Never commit secrets to Git
- Use secret templates (.tmpl files)
- Follow principle of least privilege
- Keep NFS permissions locked down

### Deployment Lessons Learned

#### NVMe Storage on pi-control
- **Always add CriticalAddonsOnly toleration** when deploying to pi-control for NVMe storage
- Pods without this toleration will fail to schedule with "node had untolerated taint" error
- This applies to: databases, monitoring hubs, and any workload needing high-speed NVMe storage

```yaml
tolerations:
- key: CriticalAddonsOnly
  operator: Equal
  value: "true"
  effect: NoExecute
```

#### K3s-Specific Paths
- **Containerd socket:** `/run/k3s/containerd/containerd.sock` (NOT `/var/run/containerd/containerd.sock`)
- This path is required for DaemonSets that need container runtime access (like monitoring agents)
- Using wrong path causes ContainerCreating failures

#### NetworkPolicy Best Practices
- **Namespace labels:** Use `kubernetes.io/metadata.name: <namespace-name>`, not `name: <namespace-name>`
- **Always verify namespace labels** with `kubectl get namespace <name> --show-labels` before creating NetworkPolicy rules
- **Cloudflare Tunnel ingress:** Must allow ingress from cloudflare namespace using correct label
- **Host network services:** For pods using `hostNetwork: true`, allow egress to node IP range (e.g., 10.0.0.0/24)

Example NetworkPolicy for Cloudflare Tunnel access:
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: cloudflare
  ports:
  - protocol: TCP
    port: 8090
```

#### Beszel Monitoring Specific
- **Mount path:** `/beszel_data` (underscore, not `/beszel/data`)
- **SSH keys:** Store in Kubernetes secrets, reference via `secretKeyRef`
- **Agent connectivity:** Hub connects TO agents on node IPs via SSH (reverse of typical monitoring)
- **NetworkPolicy:** Must allow hub egress to 10.0.0.0/24 on port 45876

#### Database Backup Considerations
- **MySQL root user:** Often restricted to localhost; use application user instead
- **Direct NFS mounts:** Preferred over PVCs for backups to match secrets backup location
- **pg_dumpall vs pg_dump:** Use `pg_dumpall` to backup all databases + roles in single file
- **Staggered schedules:** Offset backup times (e.g., PostgreSQL at 2:00 AM, MySQL at 2:30 AM)
- **Retention policies:** Dev (14 days), Prod (30 days) based on data criticality

#### Redis Stack Specific
- **Redis Insight UI:** The `redis/redis-stack` image requires the default entrypoint to start both Redis and Insight
- **DO NOT override command:** Using `command: redis-stack-server` only starts Redis, not the Insight UI
- **Use REDIS_ARGS env var:** Configure Redis via `REDIS_ARGS` environment variable instead of command override
- **Port 8001:** Redis Insight UI runs on port 8001 as a Node.js process alongside Redis
- **Verify both services:** Check `ps aux` in container - should see both redis-server and node processes

#### MetalLB LoadBalancer Best Practices
- **Label selectors:** Verify pod labels match service selectors (e.g., `app.kubernetes.io/name` vs `app`)
- **Check endpoints:** Use `kubectl get endpoints` to verify LoadBalancer services have endpoints
- **Static IP assignment:** Use annotation `metallb.universe.tf/loadBalancerIPs: 10.0.0.X`
- **Testing from control plane:** LoadBalancer IPs may not be accessible from the same node (routing issue)
- **Test from external device:** Verify LoadBalancer access from LAN devices, not cluster nodes
- **Tailscale integration:** LoadBalancers on 10.0.0.0/24 automatically accessible via Tailscale subnet route

#### General Kubernetes Patterns
- **Verify cluster state first:** Run `kubectl get nodes`, `kubectl get pods -A` before deployments
- **Check for existing resources:** Look for old installations that may conflict
- **Read existing implementations:** Check similar services for proven patterns
- **Test manually before CronJobs:** Create test jobs with `kubectl create job --from=cronjob/...`
- **Document as you go:** Update CLAUDE_CONTEXT.md with deployment-specific gotchas immediately

---

## Troubleshooting

### "Connection refused" Errors
```bash
# Check if service exists
kubectl get svc -n <namespace>

# Check if pods are running
kubectl get pods -n <namespace>

# Check if port-forward is active
ps aux | grep kubectl | grep port-forward

# Restart port-forward if needed
kubectl port-forward -n dev svc/postgresql-dev 5432:5432 &
```

### Pod Won't Start
```bash
# Check pod events
kubectl describe pod <pod-name> -n <namespace>

# Common issues:
# - ImagePullBackOff: Check image exists for ARM64
# - CrashLoopBackOff: Check logs for application errors
# - Pending: Check resource availability or storage issues

# View logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous  # Previous crashed container
```

### Database Connection Issues
```bash
# Test from debug pod
kubectl run debug --rm -it --image=busybox -n dev -- sh
# Then inside pod:
nc -zv postgresql-dev.dev.svc.cluster.local 5432
```

### NFS Permission Issues
```bash
# On NAS, check export permissions
# On pod, check if user has write access:
kubectl exec -it <pod-name> -n <namespace> -- ls -la /data

# If needed, fix on NAS:
ssh admin@10.0.0.5
sudo chown -R 1000:1000 /path/to/nfs/export
sudo chmod -R 775 /path/to/nfs/export
```

### kubectl Not Working on pi-worker-1
```bash
# Re-copy kubeconfig
scp pi@10.0.0.10:/etc/rancher/k3s/k3s.yaml ~/.kube/config

# Update server address
sed -i 's/127.0.0.1/10.0.0.10/g' ~/.kube/config

# Set permissions
chmod 600 ~/.kube/config

# Test
kubectl get nodes
```

---

## Quick Reference Card

### Port-Forward All Dev Services
```bash
# PostgreSQL
kubectl port-forward -n dev svc/postgresql-dev 5432:5432 &

# Gitea
kubectl port-forward -n dev svc/gitea-http 3000:3000 &
```

### Get All Database Passwords
```bash
# PostgreSQL Dev
kubectl get secret postgresql-dev -n dev -o jsonpath='{.data.postgres-password}' | base64 -d

# MySQL (Ghost)
kubectl get secret mysql-ghost -n prod -o jsonpath='{.data.root-password}' | base64 -d
```

### Deploy New Application
```bash
# Build image
docker buildx build --platform linux/arm64 -t myapp:v1 .

# Push to registry
docker push myapp:v1

# Apply manifests
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Check status
kubectl get pods -n <namespace>
kubectl logs -f <pod-name> -n <namespace>
```

### Common Health Checks
```bash
# Cluster health
kubectl get nodes
kubectl get pods -A
kubectl top nodes

# Service health
kubectl get svc -A
kubectl get ingress -A

# Storage health
kubectl get pv
kubectl get pvc -A

# External access
curl -I https://blog.yourdomain.com
tailscale status
```

---

## Project Documentation

- **Repository:** `~/pi-cluster-infrastructure` (on pi-control)
- **Original Implementation Plan for high-level context:** `docs/implementation_plan/simplified-implementation-plan.md`
  - This document is not neccessarily up to date with active configurations defined in the *.yaml files
- **Deviations:** `docs/DEVIATIONS.md`
- **Storage Strategy:** `docs/storage-strategy-nvme.md`
- **Secrets Strategy:** `secrets/secrets-strategy.md`
- **Current State Analysis:** `docs/cluster-state-analysis.md`

---

## External Resources

- **K3s Documentation:** https://docs.k3s.io
- **kubectl Cheat Sheet:** https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **Traefik Documentation:** https://doc.traefik.io/traefik/
- **PostgreSQL Documentation:** https://www.postgresql.org/docs/
- **Gitea Documentation:** https://docs.gitea.com/

---

**This context file is maintained to reflect the actual state of the cluster.**
**Last verified:** 2025-11-27 against live cluster state

**Recent Updates:**
- Enabled Redis caching for Ghost Blog (2025-11-27)
  - Added cache configuration to Ghost deployment via environment variables
  - Duplicated `redis-prod-credentials` secret to `ghost` namespace (cross-namespace requirement)
  - Ghost now uses Redis DB 0 for caching page fragments, queries, and API responses
- Added Redis Cache (PROD) and Redis Stack (DEV) deployments (2025-11-27)
- Configured MetalLB LoadBalancers for PostgreSQL and Redis Stack with static IPs (2025-11-27)
- Added Beszel monitoring stack (2025-11-27)
- Implemented automated database backups for PostgreSQL and MySQL (2025-11-27)
- Added comprehensive deployment lessons learned section (2025-11-27)
