# Cloudflare Tunnel Setup: From Host Installation to Kubernetes-Native

## Initial Approach: Direct Host Installation

The original plan was to install `cloudflared` directly on the pi-control node as a systemd service. This followed the traditional Cloudflare documentation approach:

1. Install the `cloudflared` binary on the host
2. Authenticate with `cloudflared tunnel login`
3. Create tunnel via CLI: `cloudflared tunnel create pi-cluster`
4. Store credentials in `/root/.cloudflared/` and config in `/etc/cloudflared/`
5. Run as a systemd service

**Problems with this approach:**
- Configuration files stored on host filesystem
- Credentials in plain text files on disk
- Single point of failure (only on control node)
- Requires SSH access to update configuration
- Not managed by Kubernetes (separate monitoring/restart mechanism)
- Manual backup and disaster recovery

## Better Approach: Kubernetes-Native Deployment

After recognizing the limitations, we pivoted to a fully Kubernetes-native approach that aligns with infrastructure-as-code principles.

### Key Strategy Changes

#### 1. **Tunnel Creation: Dashboard vs CLI**

**Old way:** CLI-based tunnel creation
```bash
cloudflared tunnel login
cloudflared tunnel create pi-cluster
```

**New way:** Cloudflare Zero Trust dashboard
- Navigate to Access → Tunnels → Create tunnel
- Copy the tunnel token (starts with `eyJ...`)
- No local CLI installation needed

**Why better:**
- Token-based authentication (more secure than cert files)
- No host-level dependencies
- Easier for team collaboration
- Token can be rotated without reinstalling

#### 2. **Credential Storage: Files vs Kubernetes Secrets**

**Old way:** Credentials stored in files
```bash
/root/.cloudflared/<tunnel-id>.json
/etc/cloudflared/config.yml
```

**New way:** Kubernetes secrets
```bash
kubectl create secret generic cloudflare-tunnel-token \
  --from-literal=token=<your-token> \
  -n cloudflare
```

**Why better:**
- Centralized secret management
- Encrypted at rest (if configured)
- RBAC controls who can access
- Easy to rotate/update
- Backed up with cluster state

#### 3. **Configuration: Static Files vs Dashboard**

**Old way:** Configuration in YAML file
```yaml
# /etc/cloudflared/config.yml
tunnel: <tunnel-id>
credentials-file: /root/.cloudflared/<tunnel-id>.json
ingress:
  - hostname: domain.com
    service: http://traefik:80
```

**New way:** Configure via Cloudflare dashboard
- Public Hostname tab in tunnel settings
- Point-and-click route configuration
- No config file to manage

**Why better:**
- Change routes without redeploying pods
- No configuration drift
- Visual interface for non-technical users
- Instant updates without pod restarts

#### 4. **High Availability: Single Service vs Replicas**

**Old way:** Single systemd service on control node
```bash
sudo systemctl start cloudflared
```

**New way:** Multiple pod replicas
```yaml
spec:
  replicas: 2
```

**Why better:**
- Survives node failures
- Load distributed across cluster
- Kubernetes auto-restarts failed pods
- Can scale up during high traffic

### Lessons Learned

#### Lesson 1: Health Checks Need Validation

**The Problem:**
Initial deployment included liveness probes checking port 2000:
```yaml
livenessProbe:
  httpGet:
    path: /ready
    port: 2000
```

But cloudflared's metrics server runs on port 20241, causing continuous pod crashes:
```
Liveness probe failed: dial tcp 10.42.2.6:2000: connect: connection refused
Container cloudflared failed liveness probe, will be restarted
```

**The Fix:**
Remove health checks entirely - they're optional for cloudflared:
```yaml
# No liveness or readiness probes needed
```

**Takeaway:** Always verify health check endpoints match actual application ports. Don't blindly copy probe configurations.

#### Lesson 2: Logs Tell the Real Story

The key to debugging was reading the pod logs:
```bash
kubectl logs -n cloudflare -l app=cloudflared --tail=100 --previous
```

Critical insights from logs:
- ✅ `Registered tunnel connection` - tunnel was actually working
- ⚠️ `Initiating graceful shutdown due to signal terminated` - being killed by Kubernetes
- ℹ️ `Starting metrics server on [::]:20241/metrics` - revealed actual port

**Takeaway:** When pods crash, use `kubectl logs --previous` to see what happened before the restart.

#### Lesson 3: Error Messages Can Be Misleading

Saw scary-looking errors in logs:
```
ERR Cannot determine default origin certificate path...
```

**Reality:** This is a harmless warning when using token-based auth. The tunnel worked fine.

**Takeaway:** Not all ERR messages are actual errors. Check if the service is functioning despite warnings.

#### Lesson 4: Infrastructure as Code Benefits

Organizing manifests in a Git repository structure:
```
~/pi-cluster-infrastructure/
├── networking/
│   ├── cloudflare/
│   │   └── cloudflared-deployment.yaml
│   ├── metallb/
│   └── traefik/
├── storage/
├── monitoring/
└── applications/
```

**Benefits realized:**
- Version control for all cluster configuration
- Easy rollback to previous versions
- Documentation through commit messages
- Shareable with team or community
- Disaster recovery: `git clone` + `kubectl apply`

#### Lesson 5: Simplicity Wins

**Final working deployment:** 35 lines of YAML
- No config files
- No health checks (optional)
- No service definition (unused)
- Just: namespace, deployment, secret reference

**Takeaway:** Start minimal, add complexity only when needed.

### Migration Path for Others

If you're currently running cloudflared on a host and want to migrate:

1. **Create tunnel via dashboard** (or keep existing tunnel)
2. **Get/regenerate tunnel token** from Cloudflare dashboard
3. **Deploy to Kubernetes** with the clean YAML
4. **Verify tunnel shows "Healthy"** in dashboard
5. **Test traffic flows** through new pods
6. **Stop old systemd service:**
```bash
   sudo systemctl stop cloudflared
   sudo systemctl disable cloudflared
```
7. **Clean up old files:**
```bash
   sudo rm -rf /etc/cloudflared
   rm -rf ~/.cloudflared
```

### Final Architecture

**Request flow:**
```
Internet User
    ↓
Cloudflare Edge Network (CDN, DDoS protection, SSL)
    ↓
Cloudflare Tunnel (encrypted)
    ↓
cloudflared pods (2 replicas in cluster)
    ↓
Traefik Ingress Controller
    ↓
Application Services
```

**Key advantages:**
- ✅ Zero exposed ports on home network
- ✅ Automatic HTTPS/SSL
- ✅ DDoS protection
- ✅ High availability (2+ tunnel replicas)
- ✅ Kubernetes-native (auto-restart, scaling)
- ✅ Secrets managed properly
- ✅ Infrastructure as code
- ✅ Easy to update/rollback

### Conclusion

The shift from host-based installation to Kubernetes-native deployment represents a maturation in infrastructure management. While the initial CLI approach seemed simpler, the K8s approach provides:

- **Better security** through proper secret management
- **Higher reliability** through replication and auto-restart
- **Easier operations** through declarative configuration
- **Team scalability** through GitOps workflows

For production homelab clusters, the small additional complexity of Kubernetes manifests pays dividends in maintainability and resilience.