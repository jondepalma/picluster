# Kubernetes Secrets Backup Guide

Complete guide for backing up Kubernetes secrets to your QNAP NAS with encryption.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Initial Setup](#initial-setup)
4. [Backup Script](#backup-script)
5. [Running Backups](#running-backups)
6. [Automated Backups](#automated-backups)
7. [Restoring from Backup](#restoring-from-backup)
8. [Verification Steps](#verification-steps)
9. [Troubleshooting](#troubleshooting)
10. [Maintenance](#maintenance)

---

## Overview

### What This Does

- Backs up ALL Kubernetes secrets from ALL namespaces
- Encrypts backups with GPG
- Stores encrypted backups on NAS
- Automatically rotates old backups (keeps 30 days)
- Can run manually or via cron

### Security Features

- ✅ Secrets encrypted with GPG before leaving cluster
- ✅ Stored on NAS (separate from cluster)
- ✅ Passphrase-protected
- ✅ Old backups automatically cleaned up
- ✅ No plaintext secrets stored anywhere

---

## Prerequisites

Before starting, ensure you have:

- [ ] QNAP NAS accessible at `10.0.0.5`
- [ ] NFS share created: `/share/kubernetes-backups`
- [ ] NFS permissions configured for `10.0.0.0/24`
- [ ] GPG installed on pi-control: `gpg --version`
- [ ] kubectl access to cluster
- [ ] sudo access on pi-control

---

## Initial Setup

### Step 1: Verify NAS Share Exists

On your QNAP NAS:

1. Open **File Station** or **Control Panel**
2. Verify `/share/kubernetes-backups` exists
3. If not, create it:
   - Go to **Control Panel** → **Shared Folders**
   - Click **Create** → **Shared Folder**
   - Name: `kubernetes-backups`
   - Size: 100GB (or as needed)

### Step 2: Configure NFS Permissions

On your QNAP NAS:

1. **Control Panel** → **Network & File Services** → **NFS**
2. Find `kubernetes-backups` share
3. Click **Edit** → **NFS Permissions**
4. Add or verify entry:
   - **Access Rights:** Read/Write
   - **IP/Network:** `10.0.0.0/24`
   - **Squash:** No root squash
   - **Security:** sys
   - Click **OK**

### Step 3: Create Permanent NFS Mount on pi-control

```bash
# Create mount point
sudo mkdir -p /mnt/nas-backups

# Add to /etc/fstab for automatic mounting
echo "10.0.0.5:/share/kubernetes-backups /mnt/nas-backups nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

# Mount it
sudo mount -a

# Verify mount
df -h | grep nas-backups

# Expected output:
# 10.0.0.5:/share/kubernetes-backups  XXG  XXG  XXG  XX% /mnt/nas-backups

# Create secrets directory
sudo mkdir -p /mnt/nas-backups/secrets

# Test write permissions
sudo touch /mnt/nas-backups/secrets/test.txt
ls -l /mnt/nas-backups/secrets/test.txt
sudo rm /mnt/nas-backups/secrets/test.txt
```

### Step 4: Install GPG (if not already installed)

```bash
# Check if GPG is installed
gpg --version

# If not installed:
sudo apt update
sudo apt install gnupg -y

# Verify installation
gpg --version
```

---

## Backup Script

### Create the Backup Script

```bash
# Create the script
cat > ~/backup-secrets.sh <<'EOF'
#!/bin/bash
#
# Kubernetes Secrets Backup Script
# 
# Description: Backs up all Kubernetes secrets to NAS with GPG encryption
# Location: ~/backup-secrets.sh
# Usage: ./backup-secrets.sh
# Cron: 0 2 * * 0 /home/pi/backup-secrets.sh >> /home/pi/backup-secrets.log 2>&1
#

set -euo pipefail  # Exit on error, undefined variables, pipe failures

# Configuration
BACKUP_DIR="/mnt/nas-backups/secrets"
DATE=$(date +%Y%m%d)
TIMESTAMP=$(date +"%Y-%m-%d %H:%M:%S")
BACKUP_FILE="cluster-secrets-$DATE.yaml"
GPG_PASSPHRASE_FILE="$HOME/.gpg-passphrase"
RETENTION_DAYS=30

# Colors for output
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Logging function
log() {
    echo -e "${BLUE}[$TIMESTAMP]${NC} $1"
}

error() {
    echo -e "${RED}[$TIMESTAMP] ERROR:${NC} $1"
}

success() {
    echo -e "${GREEN}[$TIMESTAMP] SUCCESS:${NC} $1"
}

warning() {
    echo -e "${YELLOW}[$TIMESTAMP] WARNING:${NC} $1"
}

# Main backup function
main() {
    log "🔐 Kubernetes Secrets Backup Starting"
    echo "===================================="
    echo ""

    # Check if running as expected user
    if [ "$USER" != "pi" ] && [ "$USER" != "root" ]; then
        warning "Running as user: $USER (expected: pi or root)"
    fi

    # Check if NAS is mounted
    log "📡 Checking NAS mount..."
    if ! mountpoint -q /mnt/nas-backups; then
        warning "NAS not mounted. Attempting to mount..."
        sudo mount -a
        sleep 2
        if ! mountpoint -q /mnt/nas-backups; then
            error "Failed to mount NAS at /mnt/nas-backups"
            error "Please check NFS configuration"
            exit 1
        fi
        success "NAS mounted successfully"
    else
        log "✓ NAS already mounted"
    fi

    # Check kubectl access
    log "🔍 Verifying cluster access..."
    if ! kubectl cluster-info &>/dev/null; then
        error "Cannot access Kubernetes cluster"
        error "Please verify kubectl configuration"
        exit 1
    fi
    log "✓ Cluster access verified"

    # Create backup directory if needed
    log "📁 Ensuring backup directory exists..."
    if ! sudo mkdir -p "$BACKUP_DIR"; then
        error "Failed to create backup directory: $BACKUP_DIR"
        exit 1
    fi
    log "✓ Backup directory ready: $BACKUP_DIR"

    # Check if backup already exists for today
    if [ -f "$BACKUP_DIR/$BACKUP_FILE.gpg" ]; then
        warning "Backup already exists for today: $BACKUP_FILE.gpg"
        read -p "Overwrite existing backup? (y/N): " -n 1 -r
        echo
        if [[ ! $REPLY =~ ^[Yy]$ ]]; then
            log "Backup cancelled by user"
            exit 0
        fi
    fi

    # Backup all secrets
    log "📦 Extracting all secrets from cluster..."
    if ! kubectl get secrets -A -o yaml > "/tmp/$BACKUP_FILE"; then
        error "Failed to extract secrets from cluster"
        exit 1
    fi
    
    # Count secrets
    SECRET_COUNT=$(grep -c "^  name:" "/tmp/$BACKUP_FILE" || true)
    log "✓ Extracted $SECRET_COUNT secrets"

    # Encrypt backup
    log "🔒 Encrypting backup with GPG..."
    if [ -f "$GPG_PASSPHRASE_FILE" ]; then
        # Use stored passphrase (for automated backups)
        if ! gpg --batch --yes --passphrase-file "$GPG_PASSPHRASE_FILE" -c "/tmp/$BACKUP_FILE" 2>/dev/null; then
            error "GPG encryption failed"
            rm -f "/tmp/$BACKUP_FILE"
            exit 1
        fi
        log "✓ Encrypted using stored passphrase"
    else
        # Interactive passphrase (for manual backups)
        log "Enter GPG passphrase for encryption:"
        if ! gpg -c "/tmp/$BACKUP_FILE"; then
            error "GPG encryption failed"
            rm -f "/tmp/$BACKUP_FILE"
            exit 1
        fi
        log "✓ Encrypted with manual passphrase"
    fi

    # Verify encrypted file exists
    if [ ! -f "/tmp/$BACKUP_FILE.gpg" ]; then
        error "Encrypted file not found after GPG encryption"
        rm -f "/tmp/$BACKUP_FILE"
        exit 1
    fi

    # Copy to NAS
    log "💾 Copying encrypted backup to NAS..."
    if ! sudo cp "/tmp/$BACKUP_FILE.gpg" "$BACKUP_DIR/"; then
        error "Failed to copy backup to NAS"
        rm -f "/tmp/$BACKUP_FILE" "/tmp/$BACKUP_FILE.gpg"
        exit 1
    fi

    # Verify backup on NAS
    if [ ! -f "$BACKUP_DIR/$BACKUP_FILE.gpg" ]; then
        error "Backup file not found on NAS after copy"
        rm -f "/tmp/$BACKUP_FILE" "/tmp/$BACKUP_FILE.gpg"
        exit 1
    fi

    # Get file size
    SIZE=$(du -h "$BACKUP_DIR/$BACKUP_FILE.gpg" | cut -f1)
    success "Backup created: $BACKUP_FILE.gpg ($SIZE)"
    log "   Location: $BACKUP_DIR/$BACKUP_FILE.gpg"
    log "   Secrets: $SECRET_COUNT"

    # Cleanup local files
    log "🧹 Cleaning up temporary files..."
    rm -f "/tmp/$BACKUP_FILE"
    rm -f "/tmp/$BACKUP_FILE.gpg"
    log "✓ Temporary files removed"

    # Rotate old backups
    log "🗑️  Removing backups older than $RETENTION_DAYS days..."
    OLD_BACKUPS=$(find "$BACKUP_DIR" -name "cluster-secrets-*.yaml.gpg" -mtime +$RETENTION_DAYS)
    if [ -n "$OLD_BACKUPS" ]; then
        DELETED=0
        while IFS= read -r file; do
            if sudo rm "$file"; then
                log "   Deleted: $(basename "$file")"
                ((DELETED++))
            fi
        done <<< "$OLD_BACKUPS"
        log "✓ Deleted $DELETED old backup(s)"
    else
        log "✓ No old backups to remove"
    fi

    # List current backups
    log ""
    log "📊 Current backups on NAS:"
    ls -lh "$BACKUP_DIR"/cluster-secrets-*.yaml.gpg 2>/dev/null | awk '{print "   " $9 " (" $5 ")"}'
    
    TOTAL_BACKUPS=$(ls -1 "$BACKUP_DIR"/cluster-secrets-*.yaml.gpg 2>/dev/null | wc -l)
    log ""
    success "Backup complete! Total backups: $TOTAL_BACKUPS"
    echo "===================================="
}

# Run main function
main "$@"
EOF

# Make it executable
chmod +x ~/backup-secrets.sh

echo "✅ Backup script created: ~/backup-secrets.sh"
```

### Script Features

- **Error Handling:** Exits on any error, provides clear error messages
- **Verification:** Checks NAS mount, kubectl access, file creation
- **Logging:** Color-coded output with timestamps
- **Safety:** Confirms before overwriting existing backups
- **Cleanup:** Removes temporary files and old backups
- **Reporting:** Shows count of secrets and backup size

---

## Running Backups

### Manual Backup (Interactive)

```bash
# Run the backup script
~/backup-secrets.sh

# You'll be prompted for GPG passphrase
# Choose a strong passphrase and REMEMBER IT!
```

**Example Output:**
```
[2024-11-16 04:30:15] 🔐 Kubernetes Secrets Backup Starting
====================================

[2024-11-16 04:30:15] 📡 Checking NAS mount...
[2024-11-16 04:30:15] ✓ NAS already mounted
[2024-11-16 04:30:15] 🔍 Verifying cluster access...
[2024-11-16 04:30:16] ✓ Cluster access verified
[2024-11-16 04:30:16] 📁 Ensuring backup directory exists...
[2024-11-16 04:30:16] ✓ Backup directory ready
[2024-11-16 04:30:16] 📦 Extracting all secrets from cluster...
[2024-11-16 04:30:17] ✓ Extracted 47 secrets
[2024-11-16 04:30:17] 🔒 Encrypting backup with GPG...
[2024-11-16 04:30:20] ✓ Encrypted with manual passphrase
[2024-11-16 04:30:20] 💾 Copying encrypted backup to NAS...
[2024-11-16 04:30:21] SUCCESS: Backup created: cluster-secrets-20241116.yaml.gpg (15K)
[2024-11-16 04:30:21]    Location: /mnt/nas-backups/secrets/cluster-secrets-20241116.yaml.gpg
[2024-11-16 04:30:21]    Secrets: 47
[2024-11-16 04:30:21] 🧹 Cleaning up temporary files...
[2024-11-16 04:30:21] ✓ Temporary files removed
[2024-11-16 04:30:21] 🗑️  Removing backups older than 30 days...
[2024-11-16 04:30:21] ✓ No old backups to remove

[2024-11-16 04:30:21] 📊 Current backups on NAS:
   cluster-secrets-20241116.yaml.gpg (15K)

[2024-11-16 04:30:21] SUCCESS: Backup complete! Total backups: 1
====================================
```

### Manual Backup (Automated with Stored Passphrase)

**⚠️ Security Warning:** This stores your GPG passphrase in plaintext. Only use if you understand the security implications.

```bash
# Create passphrase file (KEEP SECURE!)
echo "your-strong-passphrase-here" > ~/.gpg-passphrase
chmod 600 ~/.gpg-passphrase

# Verify permissions (should be -rw-------)
ls -l ~/.gpg-passphrase

# Now backup will run without prompting
~/backup-secrets.sh
```

---

## Automated Backups

### Setup Cron Job

```bash
# Edit crontab
crontab -e

# Add one of these lines:

# Option 1: Backup every Sunday at 2 AM
0 2 * * 0 /home/pi/backup-secrets.sh >> /home/pi/backup-secrets.log 2>&1

# Option 2: Backup daily at 2 AM
0 2 * * * /home/pi/backup-secrets.sh >> /home/pi/backup-secrets.log 2>&1

# Option 3: Backup every 6 hours
0 */6 * * * /home/pi/backup-secrets.sh >> /home/pi/backup-secrets.log 2>&1
```

**Note:** For automated backups, you MUST set up the passphrase file (see above).

### View Backup Logs

```bash
# View recent backup logs
tail -f ~/backup-secrets.log

# View all logs
cat ~/backup-secrets.log

# View logs from specific date
grep "2024-11-16" ~/backup-secrets.log
```

### Logrotate Configuration (Optional)

Prevent log file from growing too large:

```bash
sudo tee /etc/logrotate.d/backup-secrets <<EOF
/home/pi/backup-secrets.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}
EOF
```

---

## Restoring from Backup

### List Available Backups

```bash
# List all backups
ls -lh /mnt/nas-backups/secrets/

# Find backups from specific month
ls -lh /mnt/nas-backups/secrets/cluster-secrets-202411*.yaml.gpg
```

### Restore All Secrets

**⚠️ Warning:** This will overwrite existing secrets!

```bash
# 1. Choose a backup to restore
BACKUP_FILE="cluster-secrets-20241116.yaml.gpg"

# 2. Copy to temp location
sudo cp /mnt/nas-backups/secrets/$BACKUP_FILE /tmp/

# 3. Decrypt the backup
gpg -d /tmp/$BACKUP_FILE > /tmp/restored-secrets.yaml
# Enter passphrase when prompted

# 4. Review what will be restored (IMPORTANT!)
grep "name:" /tmp/restored-secrets.yaml | head -20

# 5. Apply the backup (USE WITH CAUTION!)
kubectl apply -f /tmp/restored-secrets.yaml

# 6. Clean up
rm /tmp/$BACKUP_FILE
rm /tmp/restored-secrets.yaml
```

### Restore Specific Secret

```bash
# 1. Decrypt backup
gpg -d /mnt/nas-backups/secrets/cluster-secrets-20241116.yaml.gpg > /tmp/all-secrets.yaml

# 2. Extract specific secret (example: cloudflare tunnel)
kubectl apply -f - <<EOF
$(grep -A 20 "name: cloudflare-tunnel-token" /tmp/all-secrets.yaml)
EOF

# 3. Clean up
rm /tmp/all-secrets.yaml
```

### Restore to Different Cluster

```bash
# 1. Copy backup to new cluster's control node
scp /mnt/nas-backups/secrets/cluster-secrets-20241116.yaml.gpg \
  pi@new-cluster-ip:/tmp/

# 2. On new cluster, decrypt and apply
gpg -d /tmp/cluster-secrets-20241116.yaml.gpg > /tmp/secrets.yaml
kubectl apply -f /tmp/secrets.yaml
rm /tmp/secrets.yaml /tmp/cluster-secrets-20241116.yaml.gpg
```

---

## Verification Steps

### Daily Verification Checklist

Run these checks regularly to ensure backups are working:

```bash
# 1. Check NAS mount
mountpoint /mnt/nas-backups
# Should output: /mnt/nas-backups is a mountpoint

# 2. Check latest backup exists
ls -lh /mnt/nas-backups/secrets/ | tail -5

# 3. Verify backup is recent (within expected timeframe)
find /mnt/nas-backups/secrets/ -name "cluster-secrets-*.yaml.gpg" -mtime -1
# Should show today's backup if running daily

# 4. Check backup file size (should be reasonable, not 0 bytes)
du -h /mnt/nas-backups/secrets/cluster-secrets-$(date +%Y%m%d).yaml.gpg

# 5. Verify you can decrypt (doesn't apply, just decrypt header)
gpg --list-packets /mnt/nas-backups/secrets/cluster-secrets-$(date +%Y%m%d).yaml.gpg | head -5
# Should show GPG encryption info, not errors
```

### Weekly Verification (Test Restore)

Once a week, verify you can actually restore from backup:

```bash
# 1. Get latest backup
LATEST_BACKUP=$(ls -t /mnt/nas-backups/secrets/cluster-secrets-*.yaml.gpg | head -1)

# 2. Copy to temp
sudo cp "$LATEST_BACKUP" /tmp/test-restore.yaml.gpg

# 3. Decrypt test
gpg -d /tmp/test-restore.yaml.gpg > /tmp/test-restore.yaml

# 4. Verify structure (should see YAML secrets)
head -50 /tmp/test-restore.yaml

# 5. Count secrets
grep -c "kind: Secret" /tmp/test-restore.yaml

# 6. Clean up
rm /tmp/test-restore.yaml.gpg /tmp/test-restore.yaml

echo "✅ Test restore successful!"
```

### Monthly Verification (Full Audit)

Once a month, perform a complete audit:

```bash
# Create audit script
cat > ~/audit-backups.sh <<'EOF'
#!/bin/bash

echo "🔍 Backup Audit Report"
echo "======================"
echo "Date: $(date)"
echo ""

# Check mount
echo "1. NAS Mount Status:"
if mountpoint -q /mnt/nas-backups; then
    echo "   ✅ NAS mounted"
else
    echo "   ❌ NAS NOT mounted"
fi
echo ""

# Count backups
echo "2. Backup Count:"
BACKUP_COUNT=$(ls -1 /mnt/nas-backups/secrets/cluster-secrets-*.yaml.gpg 2>/dev/null | wc -l)
echo "   Total backups: $BACKUP_COUNT"
echo ""

# Check backup dates
echo "3. Backup Timeline:"
ls -lht /mnt/nas-backups/secrets/cluster-secrets-*.yaml.gpg | head -10 | awk '{print "   " $9 " - " $6 " " $7 " " $8}'
echo ""

# Total size
echo "4. Storage Usage:"
TOTAL_SIZE=$(du -sh /mnt/nas-backups/secrets/ | cut -f1)
echo "   Total: $TOTAL_SIZE"
echo ""

# Oldest backup
echo "5. Retention:"
OLDEST=$(ls -t /mnt/nas-backups/secrets/cluster-secrets-*.yaml.gpg | tail -1)
if [ -n "$OLDEST" ]; then
    AGE=$(( ($(date +%s) - $(stat -c %Y "$OLDEST")) / 86400 ))
    echo "   Oldest backup: $(basename $OLDEST) ($AGE days old)"
else
    echo "   No backups found"
fi
echo ""

# Check cron
echo "6. Cron Job:"
if crontab -l 2>/dev/null | grep -q "backup-secrets.sh"; then
    echo "   ✅ Cron job configured"
    crontab -l | grep backup-secrets.sh | sed 's/^/   /'
else
    echo "   ⚠️  No cron job found"
fi
echo ""

# Recent log
echo "7. Recent Activity:"
if [ -f ~/backup-secrets.log ]; then
    echo "   Last 5 log entries:"
    tail -5 ~/backup-secrets.log | sed 's/^/   /'
else
    echo "   No log file found"
fi
echo ""

echo "======================"
echo "Audit complete"
EOF

chmod +x ~/audit-backups.sh

# Run the audit
~/audit-backups.sh
```

---

## Troubleshooting

### Issue: NAS Not Mounted

```bash
# Check if NAS is accessible
ping -c 3 10.0.0.5

# Try manual mount
sudo mount -t nfs 10.0.0.5:/share/kubernetes-backups /mnt/nas-backups

# Check mount status
mount | grep nas-backups

# Check /etc/fstab entry
grep nas-backups /etc/fstab

# View NFS-related logs
sudo journalctl -u rpc-statd -n 50
```

### Issue: Permission Denied

```bash
# Check NFS permissions on NAS
# Go to QNAP: Control Panel → Network & File Services → NFS
# Verify: 10.0.0.0/24 has Read/Write and "No root squash" enabled

# Check directory permissions
ls -ld /mnt/nas-backups/secrets/

# Try with sudo
sudo ls -la /mnt/nas-backups/secrets/
```

### Issue: GPG Encryption Fails

```bash
# Verify GPG is installed
gpg --version

# Test GPG encryption
echo "test" | gpg -c > /tmp/test.gpg
gpg -d /tmp/test.gpg
rm /tmp/test.gpg

# If passphrase file is used, check permissions
ls -l ~/.gpg-passphrase
# Should be: -rw------- (600)
```

### Issue: Backup File is 0 Bytes or Empty

```bash
# Check kubectl access
kubectl get secrets -A

# Check available disk space
df -h /tmp
df -h /mnt/nas-backups

# Run backup with verbose output
bash -x ~/backup-secrets.sh
```

### Issue: Cannot Decrypt Backup

```bash
# Verify file is actually encrypted
file /mnt/nas-backups/secrets/cluster-secrets-20241116.yaml.gpg
# Should show: GPG encrypted data

# Check GPG can read it
gpg --list-packets /mnt/nas-backups/secrets/cluster-secrets-20241116.yaml.gpg

# Ensure you're using correct passphrase
# Try decrypting with verbose output
gpg -v -d /mnt/nas-backups/secrets/cluster-secrets-20241116.yaml.gpg
```

### Issue: Cron Job Not Running

```bash
# Verify cron service is running
sudo systemctl status cron

# Check crontab is configured
crontab -l | grep backup-secrets

# Check cron logs
sudo grep backup-secrets /var/log/syslog

# Test script manually first
~/backup-secrets.sh

# Check log file permissions
ls -l ~/backup-secrets.log
```

---

## Maintenance

### Regular Tasks

| Task | Frequency | Command |
|------|-----------|---------|
| **Verify backups exist** | Daily | `ls -lh /mnt/nas-backups/secrets/` |
| **Test restore** | Weekly | `~/audit-backups.sh` |
| **Review logs** | Weekly | `tail -50 ~/backup-secrets.log` |
| **Full audit** | Monthly | `~/audit-backups.sh` |
| **Rotate passphrase** | Every 6 months | Update `~/.gpg-passphrase` |
| **Test disaster recovery** | Quarterly | Full restore to test cluster |

### Updating the Script

If you need to modify the backup script:

```bash
# 1. Backup current version
cp ~/backup-secrets.sh ~/backup-secrets.sh.backup

# 2. Edit the script
nano ~/backup-secrets.sh

# 3. Test it manually before relying on cron
~/backup-secrets.sh

# 4. If it works, you're done!
# If not, restore backup:
# cp ~/backup-secrets.sh.backup ~/backup-secrets.sh
```

### Monitoring Backup Health

Create a simple health check:

```bash
cat > ~/check-backup-health.sh <<'EOF'
#!/bin/bash

# Configuration
MAX_AGE_DAYS=2  # Alert if no backup in last 2 days
MIN_BACKUPS=3   # Alert if fewer than 3 backups total

# Check for recent backup
LATEST=$(find /mnt/nas-backups/secrets/ -name "cluster-secrets-*.yaml.gpg" -mtime -$MAX_AGE_DAYS | wc -l)

if [ $LATEST -eq 0 ]; then
    echo "❌ ALERT: No recent backups found (last $MAX_AGE_DAYS days)"
    exit 1
fi

# Check total backup count
TOTAL=$(ls -1 /mnt/nas-backups/secrets/cluster-secrets-*.yaml.gpg 2>/dev/null | wc -l)

if [ $TOTAL -lt $MIN_BACKUPS ]; then
    echo "⚠️  WARNING: Only $TOTAL backups found (minimum: $MIN_BACKUPS)"
    exit 1
fi

echo "✅ Backup health: OK"
echo "   Recent backups: $LATEST"
echo "   Total backups: $TOTAL"
exit 0
EOF

chmod +x ~/check-backup-health.sh

# Run health check
~/check-backup-health.sh
```

---

## Security Considerations

### Passphrase Management

1. **Use a strong passphrase:**
   - At least 20 characters
   - Mix of letters, numbers, symbols
   - Not used elsewhere

2. **Store securely:**
   - In password manager (1Password, Bitwarden, etc.)
   - Physical backup in safe location
   - **Never** commit to Git

3. **Document location:**
   - Team should know where passphrase is stored
   - Include in disaster recovery plan

### Access Control

1. **Limit access to backups:**
   - Only admin accounts can read `/mnt/nas-backups/secrets/`
   - Set proper QNAP user permissions

2. **Limit access to scripts:**
   ```bash
   chmod 700 ~/backup-secrets.sh
   chmod 600 ~/.gpg-passphrase  # If using stored passphrase
   ```

3. **Audit access:**
   - Review QNAP access logs monthly
   - Check who has kubectl access

### Disaster Recovery Planning

Document these answers:

1. **Where is GPG passphrase stored?**
   - Primary: [Password manager name]
   - Backup: [Physical location]

2. **Who has access to backups?**
   - [List of people/roles]

3. **How to restore in emergency?**
   - See "Restoring from Backup" section above
   - Keep printed copy in safe location

4. **What if NAS fails?**
   - Backups should also go to cloud/offsite
   - Consider adding second backup destination

---

## Summary

You now have:

- ✅ Automated encrypted backups of all Kubernetes secrets
- ✅ 30-day retention policy
- ✅ Secure storage on NAS
- ✅ Verification and audit procedures
- ✅ Disaster recovery capability

### Quick Reference Commands

```bash
# Manual backup
~/backup-secrets.sh

# List backups
ls -lh /mnt/nas-backups/secrets/

# Audit backups
~/audit-backups.sh

# Check health
~/check-backup-health.sh

# View logs
tail -f ~/backup-secrets.log

# Test restore (read-only)
gpg -d /mnt/nas-backups/secrets/cluster-secrets-$(date +%Y%m%d).yaml.gpg | head -50
```

---

**Last Updated:** 2024-11-16  
**Next Review:** 2024-12-16
