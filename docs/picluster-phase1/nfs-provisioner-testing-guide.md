# NFS Provisioner Testing and Troubleshooting Guide

This guide covers how to test your NFS storage provisioner, monitor PVC status, and troubleshoot common issues.

## Prerequisites

- NFS server configured and accessible (QNAP NAS at 10.0.0.3)
- **NFSv4.1** enabled and verified working
- NFS subdir external provisioner installed via Helm with NFSv4.1 support
- StorageClasses created with correct provisioner name

## Quick Status Check

### Check All Storage Components

```bash
# View all StorageClasses
kubectl get storageclass

# Check NFS provisioner pod
kubectl get pods -n kube-system | grep nfs

# View all PVCs across all namespaces
kubectl get pvc -A
```

## Verify NFSv4.1 Configuration

### Check NFS Version in Use

```bash
# View active NFS mounts on worker nodes
# First, find which node the provisioner is running on
kubectl get pod -n kube-system -l app=nfs-subdir-external-provisioner -o wide

# SSH to that node and check the mount
ssh pi@<node-ip> 'mount | grep nfs'
```

**Expected output should include:**
```
10.0.0.3:/kubernetes-pv on /var/lib/kubelet/pods/.../volumes/kubernetes.io~nfs/... type nfs4 (rw,noatime,vers=4.1,...)
```

Look for `vers=4.1` to confirm NFSv4.1 is being used.

### Test NFSv4.1 Connection Manually

```bash
# On pi-control or any worker node
sudo mkdir -p /mnt/test-nfs4

# Try mounting with NFSv4.1
sudo mount -t nfs -o nfsvers=4.1 10.0.0.3:/kubernetes-pv /mnt/test-nfs4

# Check if mounted
mount | grep test-nfs4

# Test write
sudo touch /mnt/test-nfs4/test-file.txt
echo "NFSv4.1 test" | sudo tee /mnt/test-nfs4/test-file.txt

# Clean up
sudo rm /mnt/test-nfs4/test-file.txt
sudo umount /mnt/test-nfs4
sudo rmdir /mnt/test-nfs4
```

## Testing NFS Storage

### Step 1: Create a Test PVC

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs-pvc
  namespace: default
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

### Step 2: Monitor PVC Status

```bash
# Check PVC status
kubectl get pvc test-nfs-pvc

# Expected output when working:
# NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# test-nfs-pvc   Bound    pvc-xxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx     1Gi        RWX            nfs-client     10s
```

**Status meanings:**
- **Pending**: Waiting for provisioner to create volume
- **Bound**: Successfully provisioned and ready to use
- **Lost**: Volume exists but PVC was deleted

### Step 3: View Detailed PVC Information

```bash
# Get detailed information including events
kubectl describe pvc test-nfs-pvc
```

Look for the **Events** section at the bottom - this shows what's happening.

### Step 4: Create a Test Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs-pod
  namespace: default
spec:
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'echo "NFS test successful - $(date)" > /mnt/test.txt && cat /mnt/test.txt && sleep 3600']
    volumeMounts:
    - name: nfs-volume
      mountPath: /mnt
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: test-nfs-pvc
EOF
```

### Step 5: Verify the Test

```bash
# Check pod status
kubectl get pod test-nfs-pod

# View pod logs (should show "NFS test successful")
kubectl logs test-nfs-pod

# Expected output:
# NFS test successful - Mon Nov 11 04:30:00 UTC 2024
```

### Step 6: Verify on NFS Server

```bash
# SSH to your NFS server and check the files
ssh pi@10.0.0.3 'ls -lR /share/CACHEDEV5_DATA/kubernetes-pv/'

# You should see a directory for your PVC with the test.txt file

# Check NFS exports configuration (verify NFSv4 is enabled)
ssh pi@10.0.0.3 'cat /etc/exports | grep kubernetes-pv'
```

**Note:** Your QNAP NAS exports both NFSv3 and NFSv4 paths. The cluster is configured to use NFSv4.1 for better performance.

### Step 7: Cleanup Test Resources

```bash
# Delete the test pod
kubectl delete pod test-nfs-pod

# Delete the test PVC
kubectl delete pvc test-nfs-pvc

# Verify deletion
kubectl get pvc test-nfs-pvc
# Should show: Error from server (NotFound)
```

## Key Commands Reference

### PVC Management

```bash
# List all PVCs in current namespace
kubectl get pvc

# List all PVCs in all namespaces
kubectl get pvc -A

# Get detailed info about a PVC
kubectl describe pvc <pvc-name>

# Delete a PVC
kubectl delete pvc <pvc-name>

# Delete a PVC in a specific namespace
kubectl delete pvc <pvc-name> -n <namespace>
```

### PersistentVolume (PV) Management

```bash
# List all PVs
kubectl get pv

# Get detailed info about a PV
kubectl describe pv <pv-name>

# Delete a PV (be careful - this can cause data loss)
kubectl delete pv <pv-name>
```

### StorageClass Management

```bash
# List all StorageClasses
kubectl get storageclass
# or
kubectl get sc

# Get detailed info about a StorageClass
kubectl describe storageclass <name>

# Delete a StorageClass
kubectl delete storageclass <name>

# Create/Update StorageClass from file
kubectl apply -f storageclass.yaml
```

### Provisioner Management

```bash
# Check provisioner pod status
kubectl get pods -n kube-system | grep nfs

# View provisioner logs
kubectl logs -n kube-system <nfs-provisioner-pod-name>

# Check provisioner deployment
kubectl get deployment -n kube-system | grep nfs

# Restart provisioner (if needed)
kubectl rollout restart deployment -n kube-system <nfs-provisioner-deployment-name>
```

## Troubleshooting Guide

### Issue 1: PVC Stuck in "Pending" Status

**Symptoms:**
```bash
kubectl get pvc
NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-nfs-pvc   Pending                                      nfs-client     5m
```

**Diagnosis Steps:**

1. Check PVC events:
```bash
kubectl describe pvc test-nfs-pvc
```

2. Look for error messages in Events section, common ones:
   - "Waiting for a volume to be created..."
   - "failed to provision volume"
   - "error getting provisioner"

3. Verify provisioner is running:
```bash
kubectl get pods -n kube-system | grep nfs
```

4. Check provisioner logs:
```bash
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep nfs-provisioner | awk '{print $1}')
```

**Common Causes and Solutions:**

#### A. Provisioner Name Mismatch

**Check provisioner name in logs:**
```bash
kubectl logs -n kube-system <nfs-provisioner-pod> | grep "Starting provisioner"
```

Look for a line like:
```
Starting provisioner controller cluster.local/nfs-provisioner-nfs-subdir-external-provisioner
```

**Compare with StorageClass:**
```bash
kubectl get storageclass nfs-client -o yaml | grep provisioner
```

**If they don't match, update StorageClass:**
```bash
# Delete old StorageClass
kubectl delete storageclass nfs-client

# Recreate with correct provisioner name
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: cluster.local/nfs-provisioner-nfs-subdir-external-provisioner
reclaimPolicy: Retain
EOF
```

#### B. NFS Server Not Accessible

**Test NFSv4.1 connectivity from a pod:**
```bash
kubectl run -it --rm debug --image=alpine --restart=Never -- sh

# Inside the pod:
apk add nfs-utils
showmount -e 10.0.0.3
mount -t nfs -o nfsvers=4.1 10.0.0.3:/kubernetes-pv /mnt
ls /mnt
exit
```

**Check NFS server firewall (NFSv4.1 uses only port 2049):**
```bash
# Test connectivity to NFS port
nc -zv 10.0.0.3 2049

# On NFS server (if firewall is enabled)
sudo ufw status
sudo ufw allow from 10.0.0.0/24 to any port nfs
```

#### C. Provisioner Not Installed

**Install NFS provisioner with NFSv4.1 support:**
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=10.0.0.3 \
  --set nfs.path=/kubernetes-pv \
  --set storageClass.create=false \
  --set storageClass.provisionerName=cluster.local/nfs-provisioner-nfs-subdir-external-provisioner \
  --set nfs.mountOptions="{nfsvers=4.1,hard,noatime}" \
  --namespace kube-system
```

**Note:** For NFSv4, use the path `/kubernetes-pv` (relative to NFSv4 root), not the full path.

### Issue 2: Pod Can't Mount PVC

**Symptoms:**
```bash
kubectl get pods
NAME           READY   STATUS              RESTARTS   AGE
test-pod       0/1     ContainerCreating   0          2m
```

**Diagnosis:**
```bash
kubectl describe pod test-pod
```

Look for events like:
- "Unable to mount volumes"
- "MountVolume.SetUp failed"
- "timed out waiting for the condition"

**Solutions:**

1. Check if PVC is bound:
```bash
kubectl get pvc
```

2. Verify NFS mount on worker node:
```bash
ssh pi@<worker-node-ip> mount | grep nfs
```

3. Check node logs:
```bash
ssh pi@<worker-node-ip> 'sudo journalctl -u kubelet -n 100'
```

### Issue 3: Cannot Delete PVC

**Symptoms:**
```bash
kubectl delete pvc test-nfs-pvc
# PVC stays in "Terminating" status indefinitely
```

**Diagnosis:**
```bash
kubectl describe pvc test-nfs-pvc
```

Look for finalizers that are preventing deletion.

**Solutions:**

1. Check if PVC is being used by a pod:
```bash
kubectl describe pvc test-nfs-pvc | grep "Used By"
```

2. Delete pods using the PVC first:
```bash
kubectl delete pod <pod-name>
```

3. Force remove finalizers (use with caution):
```bash
kubectl patch pvc test-nfs-pvc -p '{"metadata":{"finalizers":null}}'
```

### Issue 4: StorageClass Updates Fail

**Symptoms:**
```
Error: StorageClass.storage.k8s.io "nfs-client" is invalid: provisioner: Forbidden: updates to provisioner are forbidden
```

**Solution:**

You cannot update certain fields in StorageClass. Delete and recreate:

```bash
# Check if any PVCs are using this StorageClass
kubectl get pvc -A -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,STORAGECLASS:.spec.storageClassName | grep nfs-client

# If safe to delete
kubectl delete storageclass nfs-client

# Recreate with correct configuration
kubectl apply -f storageclass.yaml
```

### Issue 5: NFS Mount Permission Denied

**Symptoms in pod logs or events:**
```
mount.nfs: access denied by server
```

**Solutions:**

1. Check NFS export permissions on QNAP:
```bash
ssh pi@10.0.0.3 'cat /etc/exports'
```

For NFSv4, you should see exports like:
```
"/share/CACHEDEV5_DATA/kubernetes-pv" 10.0.0.0/24(sec=sys,rw,sync,wdelay,insecure,no_subtree_check,no_root_squash,fsid=...)
"/share/NFSv=4/kubernetes-pv" 10.0.0.0/24(sec=sys,rw,sync,wdelay,insecure,nohide,no_subtree_check,no_root_squash,fsid=...)
```

2. Update exports if needed:
```bash
ssh pi@10.0.0.3 'sudo exportfs -ra'
```

3. Verify permissions on NFS directory:
```bash
ssh pi@10.0.0.3 'ls -ld /share/CACHEDEV5_DATA/kubernetes-pv'
```

Should be readable/writable by appropriate users.

## Verification Checklist

Use this checklist to verify your NFS setup is working correctly:

- [ ] NFS provisioner pod is running in kube-system namespace
- [ ] **NFSv4.1** is enabled and working on QNAP NAS
- [ ] StorageClass provisioner name matches what's in provisioner logs
- [ ] StorageClass includes correct mount options (nfsvers=4.1, hard, noatime)
- [ ] Test PVC transitions from Pending to Bound status
- [ ] Test pod can mount the PVC successfully
- [ ] Pod can write to the NFS volume
- [ ] Files appear on the NFS server at `/share/CACHEDEV5_DATA/kubernetes-pv/`
- [ ] Worker nodes show `vers=4.1` in mount output
- [ ] PVC and pod can be deleted cleanly
- [ ] NFS server exports are configured correctly for NFSv4
- [ ] Network connectivity to port 2049 works between nodes and NFS server

## Useful Monitoring Commands

### Watch PVC Status in Real-Time

```bash
watch kubectl get pvc
```

### Watch Pod Status

```bash
watch kubectl get pods
```

### Follow Provisioner Logs

```bash
kubectl logs -n kube-system -f $(kubectl get pods -n kube-system | grep nfs-provisioner | awk '{print $1}')
```

### Check All Storage Resources

```bash
kubectl get storageclass,pv,pvc -A
```

## Advanced Troubleshooting

### Check RBAC Permissions

```bash
# Check service account used by provisioner
kubectl get deployment -n kube-system nfs-provisioner-nfs-subdir-external-provisioner -o yaml | grep serviceAccount

# Check ClusterRole
kubectl get clusterrole | grep nfs

# Check ClusterRoleBinding
kubectl get clusterrolebinding | grep nfs
```

### Debug NFS Mount Directly on Node

```bash
# SSH to a worker node
ssh pi@<worker-ip>

# Test NFSv4.1 mount manually
sudo mkdir -p /mnt/test-nfs4
sudo mount -t nfs -o nfsvers=4.1 10.0.0.3:/kubernetes-pv /mnt/test-nfs4

# Check if mounted (should show vers=4.1)
mount | grep test-nfs4

# Verify mount details
df -h | grep nfs

# Test write
sudo touch /mnt/test-nfs4/test-file.txt
echo "NFSv4.1 test" | sudo tee /mnt/test-nfs4/test-file.txt

# Clean up
sudo rm /mnt/test-nfs4/test-file.txt
sudo umount /mnt/test-nfs4
sudo rmdir /mnt/test-nfs4
```

**Note:** For NFSv4, the path is `/kubernetes-pv` (relative to NFSv4 root), not the full `/share/CACHEDEV5_DATA/kubernetes-pv` path.

### Check Network Connectivity

```bash
# From any cluster node, test NFS server connectivity
ping -c 3 10.0.0.3

# Check if NFS port 2049 is accessible (NFSv4.1 uses only this port)
nc -zv 10.0.0.3 2049

# Check what NFS exports are available
showmount -e 10.0.0.3
```

**Note:** NFSv4.1 simplifies firewall configuration by using only port 2049, unlike NFSv3 which requires multiple ports.

## Best Practices

1. **Always check PVC status before deploying applications**
   ```bash
   kubectl get pvc -w
   ```

2. **Use meaningful names for PVCs**
   ```yaml
   name: myapp-database-data
   ```

3. **Set appropriate storage sizes**
   - Start with conservative estimates
   - Use monitoring to track actual usage
   - Plan for growth

4. **Use correct access modes**
   - `ReadWriteOnce` (RWO): Single node read/write
   - `ReadWriteMany` (RWX): Multiple nodes read/write (NFS supports this)
   - `ReadOnlyMany` (ROX): Multiple nodes read-only

5. **Choose appropriate reclaim policies**
   - `Retain`: Keep data after PVC deletion (safer, manual cleanup)
   - `Delete`: Auto-delete data with PVC (convenient, potential data loss)

6. **Monitor storage usage**
   ```bash
   kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,CLAIM:.spec.claimRef.name
   ```

7. **Regular backups**
   - NFS data is on the QNAP NAS
   - Implement regular backup strategy for `/share/CACHEDEV5_DATA/kubernetes-pv`
   - Consider using QNAP's built-in snapshot features

8. **NFSv4.1 advantages in this cluster**
   - Better performance with parallel I/O operations
   - Single port (2049) simplifies firewall rules
   - Built-in file locking (no separate lockd daemon needed)
   - Improved security options available

## Quick Reference: Common Scenarios

### Scenario: Deploy New Application with NFS Storage

```bash
# 1. Create PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
EOF

# 2. Wait for bound status
kubectl get pvc myapp-data -w

# 3. Deploy application
kubectl apply -f myapp-deployment.yaml
```

### Scenario: Clean Up Old PVCs

```bash
# List all PVCs
kubectl get pvc -A

# Delete unused PVC
kubectl delete pvc <pvc-name> -n <namespace>

# Check corresponding PV (if Retain policy)
kubectl get pv
kubectl delete pv <pv-name>  # Only if you want to remove data
```

### Scenario: Change Default StorageClass

```bash
# Remove default annotation from current default
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Set new default
kubectl patch storageclass nfs-client -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
```

## Additional Resources

- [Kubernetes Persistent Volumes Documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [NFS Subdir External Provisioner GitHub](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [StorageClass Documentation](https://kubernetes.io/docs/concepts/storage/storage-classes/)

## Notes

- This cluster uses **NFSv4.1** on a QNAP NAS (10.0.0.3) - verified and tested
- For NFSv4, paths are relative to the NFSv4 root (use `/kubernetes-pv` not `/share/CACHEDEV5_DATA/kubernetes-pv`)
- NFSv4.1 mount options in use: `nfsvers=4.1,hard,noatime`
- The QNAP NAS exports both NFSv3 and NFSv4 paths - this cluster uses NFSv4
- Default namespace is used in examples - add `-n <namespace>` for other namespaces
- Always verify PVC is bound before deploying applications that depend on it
- Keep provisioner logs available for at least 24 hours for troubleshooting
- NFSv4.1 uses only port 2049, making firewall configuration simpler than NFSv3
