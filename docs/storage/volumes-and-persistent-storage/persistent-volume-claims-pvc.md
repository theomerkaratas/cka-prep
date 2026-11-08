A **PersistentVolumeClaim (PVC)** is a user request for storage within a namespace that Kubernetes automatically matches and binds to a suitable cluster-wide **PersistentVolume (PV)** based on capacity, access modes, and selectors.

---

## 1. Architectural Relationship: PV vs. PVC

```
+-----------------------------------------------------------+
| Cluster Administrator (Cluster Scope)                     |
|   Creates PersistentVolume: pv-vol1 (1Gi, ReadWriteOnce)  |
+-----------------------------------------------------------+
                             |
                             |  1-to-1 BINDING
                             v
+-----------------------------------------------------------+
| User / Application (Namespace Scope)                      |
|   Creates PersistentVolumeClaim: myclaim (500Mi, RWO)     |
+-----------------------------------------------------------+
                             ^
                             | Mounted via volumeClaim
+-----------------------------------------------------------+
| Pod (Namespace Scope)                                     |
+-----------------------------------------------------------+
```

### Key Architectural Rules (CKA Essentials):
1. **Strict 1-to-1 Binding:** Every PVC binds to **exactly one** PV, and that PV is dedicated exclusively to that single claim.
2. **No Capacity Sharing:** If a `500Mi` PVC binds to a `1Gi` PV, the remaining `500Mi` **cannot** be utilized by another PVC. The whole PV is consumed.
3. **Namespace vs. Cluster Scope:**
   - `PersistentVolume` (PV) = **Cluster-scoped** (no namespace).
   - `PersistentVolumeClaim` (PVC) = **Namespaced** (lives inside a specific namespace).

---

## 2. Declarative Manifests: PVC and PV

### A. PersistentVolumeClaim (User Request)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### B. PersistentVolume (Admin Pool)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  awsElasticBlockStore:
    volumeID: "<volume-id>"
    fsType: ext4
  persistentVolumeReclaimPolicy: Retain # Retain, Delete, or Recycle
```

---

## 3. How Kubernetes Matches & Binds PVCs to PVs

When a PVC is created, the control loop evaluates available PVs against four criteria:

1. **Sufficient Capacity:** `PV capacity >= PVC requested capacity`.
2. **Matching Access Modes:** The PV must support the access mode requested by the PVC (e.g., `ReadWriteOnce`).
3. **StorageClass Match:** If requested, both must share the same `storageClassName`. (Empty/omitted matches standard default or manual PVs).
4. **Label Selectors (Explicit Target):** If multiple PVs match, a user can pin a specific PV using `spec.selector.matchLabels` on the PVC paired with `metadata.labels` on the PV.

> ⚠️ **Pending State Gotcha:**  
> If no matching PV exists (e.g., all PVs have insufficient capacity or mismatched access modes), the PVC enters and remains in a **`Pending`** state. Once a matching PV is created or released, Kubernetes automatically transitions the PVC to **`Bound`**.

---

## 4. PersistentVolume Reclaim Policies (`persistentVolumeReclaimPolicy`)

What happens to the physical storage and the PV when a user runs `kubectl delete pvc <claim-name>`?

| Reclaim Policy | What Happens to PV? | What Happens to Underlying Data/Disk? | Reusability |
| :--- | :--- | :--- | :--- |
| **`Retain`** *(Default manual)* | Enters `Released` state | Data is left untouched on the underlying storage | Cannot be bound by another claim until an admin manually scrubs data and resets the PV |
| **`Delete`** *(Default dynamic)* | PV object is deleted | Underlying storage asset (e.g., AWS EBS, GCP PD) is automatically terminated & deleted | Completely decommissioned |
| **`Recycle`** *(Deprecated)* | Cleans data, goes to `Available` | Ran basic `rm -rf /volume/*` via a tiny scrub pod | Re-bindable |

### Why `Recycle` is Deprecated in Modern Kubernetes:
- **Security & Portability Gaps:** It only performed a rudimentary `rm -rf`, which left filesystem metadata/inodes intact, failed on restricted permissions, and could not guarantee secure cryptographic erasure.
- **Incompatible with Cloud/Block Storage:** Cloud backends require proper detachment, volume unmapping, snapshot retention, or vendor API calls—actions that a simple `rm -rf` pod cannot perform.
- **Modern Solution:** **Dynamic Provisioning** using **`StorageClass`** and out-of-tree **CSI drivers** with `reclaimPolicy: Delete` or `Retain`.

---

## 5. Mounting a PVC inside a Pod

Once bound, consume the storage inside a Pod using `persistentVolumeClaim.claimName`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  namespace: default
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /var/www/html
          name: app-storage
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: myclaim
```

---

## 6. Inspection & Verification Commands (CKA Exam)

```bash
# Check PVC status (look for Pending vs Bound)
kubectl get pvc
kubectl get persistentvolumeclaim -n <namespace>

# Verify PV-to-PVC binding and capacity matching
kubectl get pv
# Note the 'CLAIM' column showing '<namespace>/<pvc-name>'

# Inspect failure reasons when PVC is stuck in Pending
kubectl describe pvc myclaim -n <namespace>

# Delete PVC and observe PV state change to Released (if Retain)
kubectl delete pvc myclaim
kubectl get pv
```