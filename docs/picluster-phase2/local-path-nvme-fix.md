# Fix: local-path-nvme StorageClass Configuration

**Issue Found:** 2024-11-16  
**Affects:** simplified-implementation-plan.md, quick-start-guide.md

---

## Problem

The `local-path-nvme` StorageClass was created but **pods couldn't provision volumes**. Error message:

```
failed to provision volume with StorageClass "local-path-nvme": 
config doesn't contain path /mnt/nvme/k3s-local-path-provisioner on node pi-control
```

### Root Cause

The K3s `rancher.io/local-path` provisioner **ignores** the `nodePath` parameter in the StorageClass definition. Instead, it reads paths exclusively from the ConfigMap `local-path-config` in the `kube-system` namespace.

### What Was Wrong

Both implementation guides created the StorageClass with:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-nvme
provisioner: rancher.io/local-path
parameters:
  nodePath: /mnt/nvme/k3s-local-path-provisioner  # ❌ This is ignored!
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

This parameter is **silently ignored** by the provisioner!

---

## Solution

After creating the StorageClass and directory, you **must** update the provisioner's ConfigMap.

### Step 1: Update the ConfigMap

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: kube-system
data:
  config.json: |-
    {
      "nodePathMap":[
      {
        "node":"pi-control",
        "paths":["/mnt/nvme/k3s-local-path-provisioner"]
      },
      {
        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
        "paths":["/var/lib/rancher/k3s/storage"]
      }
      ]
    }
  helperPod.yaml: |-
    apiVersion: v1
    kind: Pod
    metadata:
      name: helper-pod
    spec:
      containers:
      - name: helper-pod
        image: "rancher/mirrored-library-busybox:1.36.1"
        imagePullPolicy: IfNotPresent
  setup: |-
    #!/bin/sh
    set -eu
    mkdir -m 0777 -p "${VOL_DIR}"
    chmod 700 "${VOL_DIR}/.."
  teardown: |-
    #!/bin/sh
    set -eu
    rm -rf "${VOL_DIR}"
EOF
```

### Step 2: Restart the Provisioner

```bash
# Restart to pick up new configuration
kubectl rollout restart deployment -n kube-system local-path-provisioner

# Wait for restart to complete
kubectl rollout status deployment -n kube-system local-path-provisioner

# Verify it's running
kubectl get pods -n kube-system -l app=local-path-provisioner
```

### Step 3: Verify Configuration

```bash
# Check the ConfigMap was updated
kubectl get configmap -n kube-system local-path-config -o yaml

# Look for the nodePathMap section with pi-control entry
```

---

## Complete Setup Sequence

For future reference, here's the **correct order** to set up `local-path-nvme`:

```bash
# 1. Create the directory on pi-control
sudo mkdir -p /mnt/nvme/k3s-local-path-provisioner
sudo chown -R root:root /mnt/nvme/k3s-local-path-provisioner
sudo chmod 755 /mnt/nvme/k3s-local-path-provisioner

# 2. Create the StorageClass (without nodePath parameter - it's ignored anyway)
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-nvme
  annotations:
    storageclass.kubernetes.io/description: "High-performance NVMe storage on pi-control"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF

# 3. Update the ConfigMap (THE CRITICAL STEP)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: kube-system
data:
  config.json: |-
    {
      "nodePathMap":[
      {
        "node":"pi-control",
        "paths":["/mnt/nvme/k3s-local-path-provisioner"]
      },
      {
        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
        "paths":["/var/lib/rancher/k3s/storage"]
      }
      ]
    }
  helperPod.yaml: |-
    apiVersion: v1
    kind: Pod
    metadata:
      name: helper-pod
    spec:
      containers:
      - name: helper-pod
        image: "rancher/mirrored-library-busybox:1.36.1"
        imagePullPolicy: IfNotPresent
  setup: |-
    #!/bin/sh
    set -eu
    mkdir -m 0777 -p "${VOL_DIR}"
    chmod 700 "${VOL_DIR}/.."
  teardown: |-
    #!/bin/sh
    set -eu
    rm -rf "${VOL_DIR}"
EOF

# 4. Restart the provisioner
kubectl rollout restart deployment -n kube-system local-path-provisioner
kubectl rollout status deployment -n kube-system local-path-provisioner

# 5. Verify
kubectl get storageclass
kubectl get pods -n kube-system -l app=local-path-provisioner
```

---

## Testing

Test that the StorageClass works:

```bash
# Create a test PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nvme-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path-nvme
  resources:
    requests:
      storage: 1Gi
EOF

# Create a test pod to trigger provisioning
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-nvme-pod
  namespace: default
spec:
  nodeSelector:
    kubernetes.io/hostname: pi-control
  tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
    effect: NoExecute
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'echo "Testing NVMe storage" > /data/test.txt && cat /data/test.txt && sleep 3600']
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: test-nvme-pvc
EOF

# Wait and check
kubectl get pvc test-nvme-pvc
kubectl get pod test-nvme-pod
kubectl logs test-nvme-pod

# Check the volume was created on NVMe
ls -la /mnt/nvme/k3s-local-path-provisioner/

# Cleanup test resources
kubectl delete pod test-nvme-pod
kubectl delete pvc test-nvme-pvc
```

---

## Key Takeaways

1. **StorageClass parameters don't configure the provisioner** - they're just metadata
2. **The ConfigMap is the source of truth** for the `rancher.io/local-path` provisioner
3. **Always restart the provisioner** after ConfigMap changes
4. **Node-specific paths** require the `nodePathMap` structure in the ConfigMap
5. **Pods using NVMe storage need:**
   - `nodeSelector` to target pi-control
   - `tolerations` to run on the tainted control plane

---

## Where to Add in Documentation

Both `simplified-implementation-plan.md` and `quick-start-guide.md` should add the ConfigMap update step immediately after creating the StorageClass, before any database deployments.

**Location in simplified-implementation-plan.md:**  
Section 1.4 "Setup NFS Storage Provisioner" → After "Create Storage Classes"

**Location in quick-start-guide.md:**  
Step 12 "Install NFS Storage Provisioner" → After creating storage classes

---

**Document Status:** ✅ Issue Resolved  
**Last Updated:** 2024-11-16
