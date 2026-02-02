# Ghost Blog Migration: blog.jondepalma.net -> jondepalma.com

**Purpose:** Migrate Ghost blog from blog.jondepalma.net to the root domain jondepalma.com
**Risk Level:** Low (phased cutover with 301 redirects, full rollback plan)
**Last Updated:** 2026-02-01

---

## Executive Summary

**What's Changing:**
- `blog.jondepalma.net` -> `jondepalma.com` (root domain)
- Search engine indexing enabled (X-Robots-Tag noindex header removed)
- `www.jondepalma.com` redirects to `jondepalma.com`

**What's Staying:**
- Ghost application, database, and storage unchanged
- Cloudflare Tunnel for external access
- All existing blog content preserved
- `blog.jondepalma.net` continues working via 301 redirect

**Infrastructure:**
- Ghost deployment in `ghost` namespace
- MySQL database in `prod` namespace (`mysql-ghost`)
- Redis cache in `prod` namespace
- Cloudflare Tunnel managed via Zero Trust Dashboard
- Traefik ingress controller

---

## Prerequisites

Complete all of these before proceeding:

- [ ] Hugo -> Ghost content migration is complete
- [ ] All blog posts migrated and verified in Ghost
- [ ] All images and assets uploaded to Ghost
- [ ] Ghost theme customized to satisfaction
- [ ] RSS feed tested at blog.jondepalma.net
- [ ] Sitemap verified at blog.jondepalma.net
- [ ] Backup of Hugo site saved for reference

---

## Phase 1: Pre-Migration Backup

### 1.1 Backup Ghost MySQL Database

```bash
ssh pi@10.0.0.10

# Trigger a manual backup via the existing CronJob
kubectl create job --from=cronjob/mysql-ghost-backup \
  ghost-pre-migration-backup-$(date +%Y%m%d) -n prod

# Wait for completion
kubectl get jobs -n prod -w

# Verify backup exists on NAS
ssh admin@10.0.0.5 "ls -lh /share/kubernetes-backups/databases/mysql-ghost/ | tail -5"
```

### 1.2 Backup Ghost Content PVC

```bash
# Get the Ghost pod name
GHOST_POD=$(kubectl get pods -n ghost -l app=ghost -o jsonpath='{.items[0].metadata.name}')

# Create tar of Ghost content
kubectl exec -n ghost $GHOST_POD -- \
  tar czf /tmp/ghost-content-pre-migration.tar.gz /var/lib/ghost/content

# Copy to local machine
kubectl cp ghost/$GHOST_POD:/tmp/ghost-content-pre-migration.tar.gz \
  ./ghost-content-pre-migration-$(date +%Y%m%d).tar.gz

# Clean up temp file in pod
kubectl exec -n ghost $GHOST_POD -- rm /tmp/ghost-content-pre-migration.tar.gz
```

### 1.3 Backup Configuration Files

```bash
cd ~/pi-cluster-infrastructure

cp applications/ghost-blog/05-ghost-deployment.yaml \
   applications/ghost-blog/05-ghost-deployment.yaml.backup-$(date +%Y%m%d)

cp applications/ghost-blog/07-ghost-ingress.yaml \
   applications/ghost-blog/07-ghost-ingress.yaml.backup-$(date +%Y%m%d)
```

### 1.4 Document Current State

```bash
# Save current deployment state
kubectl get deployment ghost -n ghost -o yaml > ghost-deployment-state-$(date +%Y%m%d).yaml
kubectl get ingress ghost -n ghost -o yaml > ghost-ingress-state-$(date +%Y%m%d).yaml

# Verify current site is working
curl -I https://blog.jondepalma.net
curl -I https://blog.jondepalma.net/ghost/api/v4/admin/site/
```

---

## Phase 2: Cloudflare Configuration (Zero Trust Dashboard)

All Cloudflare changes are made through the Zero Trust Dashboard.

### 2.1 Lower DNS TTL (24 Hours Before Migration)

**Navigate to:** Cloudflare Dashboard -> DNS -> Records

1. Find the `blog.jondepalma.net` record
2. Edit -> Set TTL to **300** (5 minutes)
3. Save

### 2.2 Add Tunnel Public Hostname for jondepalma.com

**Navigate to:** Zero Trust Dashboard -> Networks -> Tunnels -> [Your Tunnel] -> Public Hostnames

**Add new public hostname:**

| Field | Value |
|-------|-------|
| **Subdomain** | _(leave empty for root domain)_ |
| **Domain** | jondepalma.com |
| **Path** | _(leave empty)_ |
| **Type** | HTTP |
| **URL** | `ghost.ghost.svc.cluster.local:2368` |

Click **Save hostname**

### 2.3 Add DNS Records

**Navigate to:** Cloudflare Dashboard -> jondepalma.com -> DNS -> Records

1. **Add/Update root domain CNAME:**
   - Type: CNAME
   - Name: `@` (or `jondepalma.com`)
   - Target: `[your-tunnel-id].cfargotunnel.com`
   - Proxy status: Proxied (orange cloud)
   - TTL: Auto

2. **Add www CNAME:**
   - Type: CNAME
   - Name: `www`
   - Target: `jondepalma.com`
   - Proxy status: Proxied (orange cloud)
   - TTL: Auto

3. **Verify DNS propagation:**
   ```bash
   dig jondepalma.com
   dig www.jondepalma.com
   nslookup jondepalma.com
   ```

### 2.4 Create Redirect Rules

**Navigate to:** Cloudflare Dashboard -> jondepalma.net -> Rules -> Redirect Rules

**Rule 1: blog.jondepalma.net -> jondepalma.com**

| Field | Value |
|-------|-------|
| **Rule name** | Redirect blog.jondepalma.net to jondepalma.com |
| **When incoming requests match** | Custom filter expression |
| **Field** | Hostname |
| **Operator** | equals |
| **Value** | `blog.jondepalma.net` |
| **Then** | Dynamic |
| **Type** | Redirect |
| **Expression** | `concat("https://jondepalma.com", http.request.uri.path)` |
| **Status code** | 301 |
| **Preserve query string** | Checked |

Click **Deploy rule**

**Navigate to:** Cloudflare Dashboard -> jondepalma.com -> Rules -> Redirect Rules

**Rule 2: www.jondepalma.com -> jondepalma.com**

| Field | Value |
|-------|-------|
| **Rule name** | Redirect www.jondepalma.com to jondepalma.com |
| **When incoming requests match** | Custom filter expression |
| **Field** | Hostname |
| **Operator** | equals |
| **Value** | `www.jondepalma.com` |
| **Then** | Dynamic |
| **Type** | Redirect |
| **Expression** | `concat("https://jondepalma.com", http.request.uri.path)` |
| **Status code** | 301 |
| **Preserve query string** | Checked |

Click **Deploy rule**

---

## Phase 3: Kubernetes Configuration Changes

### 3.1 Update Ghost Deployment URL

**File:** `applications/ghost-blog/05-ghost-deployment.yaml`

Change line 60:
```yaml
# BEFORE
- name: url
  value: "https://blog.jondepalma.net"

# AFTER
- name: url
  value: "https://jondepalma.com"
```

This controls Ghost's internal link generation, RSS feed URLs, sitemap URLs, email templates, asset URLs, and admin panel redirects.

### 3.2 Update Ghost Ingress

**File:** `applications/ghost-blog/07-ghost-ingress.yaml`

Change TLS host (line 36):
```yaml
# BEFORE
tls:
- hosts:
  - blog.jondepalma.net  # CHANGE THIS!
  secretName: ghost-tls

# AFTER
tls:
- hosts:
  - jondepalma.com
  secretName: ghost-tls
```

Change ingress rule host (line 40):
```yaml
# BEFORE
rules:
- host: blog.jondepalma.net  # CHANGE THIS!

# AFTER
rules:
- host: jondepalma.com
```

Remove X-Robots-Tag from ghost-headers Middleware (line 60):
```yaml
# BEFORE
spec:
  headers:
    customResponseHeaders:
      X-Robots-Tag: "none,noarchive,nosnippet,notranslate,noimageindex"
    frameDeny: false

# AFTER
spec:
  headers:
    frameDeny: false
```

The `X-Robots-Tag: none` header was blocking all search engine indexing. Removing it allows Google and other search engines to index jondepalma.com.

### 3.3 Update Infrastructure Health Check

**File:** `docs/gitea-workflows/infrastructure-deploy.yaml`

Change line 259:
```yaml
# BEFORE
curl -f https://blog.jondepalma.net/ghost/api/v4/admin/site/ || echo "Warning: Ghost API not responding"

# AFTER
curl -f https://jondepalma.com/ghost/api/v4/admin/site/ || echo "Warning: Ghost API not responding"
```

---

## Phase 4: Apply Kubernetes Changes

```bash
ssh pi@10.0.0.10
cd ~/pi-cluster-infrastructure

# Apply updated Ghost deployment
kubectl apply -f applications/ghost-blog/05-ghost-deployment.yaml

# Apply updated Ghost ingress
kubectl apply -f applications/ghost-blog/07-ghost-ingress.yaml

# Monitor rollout (Ghost will restart due to URL env var change)
kubectl rollout status deployment/ghost -n ghost

# Watch pod restart
kubectl get pods -n ghost -w

# Check logs for errors
kubectl logs -n ghost deployment/ghost -f --tail=100
```

**Expected behavior:** Ghost pod restarts with the new URL configuration. The restart causes a brief period where the blog is unavailable (typically under 60 seconds).

---

## Phase 5: Post-Migration Verification

### 5.1 Basic Connectivity

```bash
# Test root domain
curl -I https://jondepalma.com
# Expected: HTTP/2 200

# Test admin panel
curl -I https://jondepalma.com/ghost
# Expected: HTTP/2 301 (redirects to /ghost/#/signin)

# Test Ghost API
curl -f https://jondepalma.com/ghost/api/v4/admin/site/
# Expected: JSON response with site info
```

### 5.2 Verify X-Robots-Tag Removed

```bash
curl -I https://jondepalma.com 2>&1 | grep -i "x-robots-tag"
# Expected: No output (header should be gone)
```

### 5.3 Test Redirects

```bash
# blog.jondepalma.net -> jondepalma.com
curl -I https://blog.jondepalma.net
# Expected: HTTP/2 301, Location: https://jondepalma.com

# Path-preserving redirect
curl -I https://blog.jondepalma.net/my-blog-post/
# Expected: HTTP/2 301, Location: https://jondepalma.com/my-blog-post/

# Ghost admin redirect
curl -I https://blog.jondepalma.net/ghost
# Expected: HTTP/2 301, Location: https://jondepalma.com/ghost

# www redirect
curl -I https://www.jondepalma.com
# Expected: HTTP/2 301, Location: https://jondepalma.com

# www path-preserving
curl -I https://www.jondepalma.com/my-blog-post/
# Expected: HTTP/2 301, Location: https://jondepalma.com/my-blog-post/
```

### 5.4 Test Ghost Functions

```bash
# RSS feed
curl -I https://jondepalma.com/rss/
# Expected: HTTP/2 200, Content-Type includes xml

# Sitemap
curl -I https://jondepalma.com/sitemap.xml
# Expected: HTTP/2 200, Content-Type includes xml
```

### 5.5 Browser Verification

- [ ] Open https://jondepalma.com - homepage loads
- [ ] Click through to a blog post - post loads, images display
- [ ] Open https://jondepalma.com/ghost - admin login page loads
- [ ] Log in to admin panel
- [ ] Navigate to Settings -> General - verify Site URL shows `https://jondepalma.com`
- [ ] Create a test draft post - verify preview URL uses jondepalma.com
- [ ] Upload a test image - verify image URL uses jondepalma.com
- [ ] Check internal links in posts - they should use jondepalma.com
- [ ] Test from mobile device
- [ ] Test from external network (not on LAN)

### 5.6 Verify HTTPS

```bash
# Check certificate
echo | openssl s_client -servername jondepalma.com -connect jondepalma.com:443 2>/dev/null | openssl x509 -noout -subject -dates
```

---

## Phase 6: SEO Setup

### 6.1 Google Search Console

1. Go to https://search.google.com/search-console
2. Add property: `https://jondepalma.com`
3. Verify ownership (Cloudflare DNS TXT record or HTML tag)
4. Submit sitemap: `https://jondepalma.com/sitemap.xml`
5. Monitor for crawl errors over the following days

### 6.2 Social Media Links

Update profile URLs on:
- [ ] GitHub
- [ ] LinkedIn
- [ ] Twitter/X
- [ ] Any other platforms with blog link

### 6.3 RSS Feed

- 301 redirects from blog.jondepalma.net/rss/ should preserve existing subscribers
- Most feed readers follow 301 redirects automatically
- Monitor Ghost analytics for subscription activity

---

## Phase 7: Documentation Updates

After verifying the migration is successful, update these files:

| File | Change |
|------|--------|
| `README.md` (line 179) | `https://blog.jondepalma.net` -> `https://jondepalma.com` |
| `docs/CLAUDE_CONTEXT.md` (line 679) | `https://blog.jondepalma.net` -> `https://jondepalma.com` |
| `docs/pi-cluster-v2/pi-cluster-decommission-guide.md` (line 554) | `curl -I https://blog.jondepalma.net` -> `curl -I https://jondepalma.com` |

---

## Rollback Plan

If something goes wrong, follow these steps to revert:

### Restore Configuration Files

```bash
cd ~/pi-cluster-infrastructure

# Restore from backups
cp applications/ghost-blog/05-ghost-deployment.yaml.backup-YYYYMMDD \
   applications/ghost-blog/05-ghost-deployment.yaml

cp applications/ghost-blog/07-ghost-ingress.yaml.backup-YYYYMMDD \
   applications/ghost-blog/07-ghost-ingress.yaml

# Apply rollback
kubectl apply -f applications/ghost-blog/05-ghost-deployment.yaml
kubectl apply -f applications/ghost-blog/07-ghost-ingress.yaml

# Force restart
kubectl rollout restart deployment/ghost -n ghost

# Monitor rollback
kubectl rollout status deployment/ghost -n ghost

# Verify old domain works
curl -I https://blog.jondepalma.net
```

### Revert Cloudflare Changes

1. **Zero Trust Dashboard -> Tunnels -> Public Hostnames:**
   - Remove the `jondepalma.com` public hostname

2. **Cloudflare Dashboard -> DNS:**
   - Remove or revert the `jondepalma.com` CNAME
   - Remove the `www.jondepalma.com` CNAME
   - Restore TTL for `blog.jondepalma.net` to Auto

3. **Cloudflare Dashboard -> Rules -> Redirect Rules:**
   - Delete the `blog.jondepalma.net` redirect rule
   - Delete the `www.jondepalma.com` redirect rule

### Rollback Decision Criteria

Trigger rollback if:
- Ghost UI not accessible at jondepalma.com after 15 minutes
- Internal links all broken
- Images not loading
- Admin panel inaccessible at jondepalma.com/ghost
- Critical errors in Ghost logs
- Database connection issues

---

## Post-Migration Maintenance

### Week 1

Daily checks:
- [ ] Monitor Ghost logs: `kubectl logs -n ghost deployment/ghost --tail=50`
- [ ] Check Cloudflare Analytics for 404 errors
- [ ] Verify redirect statistics in Cloudflare
- [ ] Test core blog functionality

### Month 1

Weekly checks:
- [ ] Review traffic patterns in Cloudflare Analytics
- [ ] Check Google Search Console for crawl errors and indexing progress
- [ ] Monitor RSS subscriber activity in Ghost analytics
- [ ] Review organic search traffic trends

### Ongoing

- Keep 301 redirect rules indefinitely (SEO best practice - never remove)
- Keep Cloudflare tunnel hostnames active (no cost)
- Quarterly: Review redirect traffic to see if old domain still getting hits

---

## Files Modified by This Migration

| File | Type | Change |
|------|------|--------|
| `applications/ghost-blog/05-ghost-deployment.yaml` | Config | URL: blog.jondepalma.net -> jondepalma.com |
| `applications/ghost-blog/07-ghost-ingress.yaml` | Config | Host: blog.jondepalma.net -> jondepalma.com; Remove X-Robots-Tag |
| `docs/gitea-workflows/infrastructure-deploy.yaml` | Config | Health check URL updated |
| `README.md` | Docs | External access URL updated |
| `docs/CLAUDE_CONTEXT.md` | Docs | Ghost external access URL updated |
| `docs/pi-cluster-v2/pi-cluster-decommission-guide.md` | Docs | Verification URL updated |
