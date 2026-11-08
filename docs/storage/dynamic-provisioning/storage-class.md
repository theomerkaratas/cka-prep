A **StorageClass** provides a way for administrators to describe the "classes" of storage they offer (e.g., standard HDD vs. fast SSD, single-zone vs. replicated) and enables **Dynamic Volume Provisioning**, completely removing the need for manual, pre-allocated PersistentVolumes.

> Its kind a template of PV

---

## 1. Static Provisioning vs. Dynamic Provisioning

Understanding the operational shift between these two models is essential for CKA:

```
Static Provisioning (Manual & Inefficient):
1. Admin manually creates disk in Cloud (e.g., AWS EBS / GCE-PD)
2. Admin writes & applies PV manifest referencing disk ID
3. Developer creates PVC requesting capacity
4. Kubernetes binds PVC to matching static PV
5. Pod mounts PVC

Dynamic Provisioning (Automated via StorageClass):
1. Admin creates StorageClass once (defines provisioner & disk parameters)
2. Developer creates PVC with `storageClassName: <class-name>`
3. StorageClass Provisioner AUTOMATICALLY talks to Cloud API -> Creates disk
4. StorageClass Provisioner AUTOMATICALLY creates the PV object
5. PVC binds to the newly created PV -> Pod mounts it immediately
```

| Dimension | Static Provisioning | Dynamic Provisioning (StorageClass) |
| :--- | :--- | :--- |
| **Cloud Disk Creation** | Manual (via Cloud Console or Terraform) | **Fully automated** on-demand |
| **PV Manifest** | Admin must manually write and apply YAML | **Auto-generated** by the provisioner |
| **Capacity Waste** | High (fixed sizes carved upfront) | Zero (provisions the exact size requested in PVC) |
| **Scaling & Multi-Tenancy** | Slow, administrative bottleneck | Fast, self-service for developers |

---

## 2. Anatomy of Dynamic Provisioning: The Three-Object Chain

Based on your diagram, dynamic provisioning links three declarative manifests together:

```
+-----------------------------------------------------------+
| 1. StorageClass: google-storage (Cluster-Scoped)          |
|    provisioner: kubernetes.io/gce-pd                      |
+-----------------------------------------------------------+
                             ^
                             | (Linked via storageClassName)
+-----------------------------------------------------------+
| 2. PersistentVolumeClaim: myclaim (Namespace-Scoped)      |
|    spec:                                                  |
|      storageClassName: google-storage                     |
|      resources.requests.storage: 500Mi                    |
+-----------------------------------------------------------+
                             ^
                             | (Mounted via claimName)
+-----------------------------------------------------------+
| 3. Pod: random-number-generator                           |
|    spec:                                                  |
|      volumes:                                             |
|        - name: data-volume                                |
|          persistentVolumeClaim:                           |
|            claimName: myclaim                             |
+-----------------------------------------------------------+
```

### Complete Declarative Manifests

#### Step 1: StorageClass Manifest (`sc-definition.yaml`)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage            # Cluster-wide name referenced by PVCs
provisioner: kubernetes.io/gce-pd  # The driver that talks to the storage backend
volumeBindingMode: Immediate     # Immediate (default) or WaitForFirstConsumer
reclaimPolicy: Delete             # Delete (default) or Retain
allowVolumeExpansion: true        # Allows increasing PVC storage size later
parameters:
  type: pd-standard               # Options: pd-standard | pd-ssd
  replication-type: none          # Options: none | regional-pd
```

#### Step 2: PersistentVolumeClaim Manifest (`pvc-definition.yaml`)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: google-storage # Tells k8s to trigger the StorageClass provisioner
  resources:
    requests:
      storage: 500Mi
```

#### Step 3: Pod Manifest (`pod-definition.yaml`)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
  namespace: default
spec:
  containers:
    - name: alpine
      image: alpine
      command: ["/bin/sh", "-c"]
      args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
      volumeMounts:
        - name: data-volume
          mountPath: /opt
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: myclaim         # Attaches the dynamically created volume
```

---

## 3. Tiers of Service: Why It Is Called a Storage "Class"

Administrators create different StorageClasses tailored for different performance, durability, and cost profiles (e.g., Bronze/Silver/Gold or Dev/Prod):

```yaml
# Silver: Cheap Standard Magnetic/HDD Disks (For logs, batch jobs)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: silver
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: none
---
# Gold: Fast SSD Disks (For databases, latency-critical apps)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gold
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: none
---
# Platinum: Regional Replicated SSDs (For high-availability & disaster recovery)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: platinum
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
```

---

## 4. Key StorageClass Parameters & Critical CKA Gotchas

### A. `volumeBindingMode` (Exam Trap!)
- **`Immediate` (Default):** As soon as the PVC is created, the volume is provisioned on the cloud backend and a PV is created—**even if no Pod uses it yet**.
  - *Gotcha:* In multi-zone clusters, a cloud volume (like AWS EBS or GCE PD) is tied to a specific Availability Zone (e.g., `us-central1-a`). If `Immediate` provisions the volume in Zone A, but the Scheduler places the Pod in Zone B, the Pod will fail with `VolumeZoneMismatch`!
- **`WaitForFirstConsumer` (Recommended for Zone-Aware Volumes):** Delays volume creation and PV binding until a Pod referencing the PVC is actually created and scheduled to a Node. The volume is then created in the exact zone where the Pod lands.

### B. `reclaimPolicy`
- Controls what happens to the underlying cloud disk and PV when the PVC is deleted.
- Defaults to **`Delete`** for dynamically provisioned volumes (can be set to `Retain`).

### C. `parameters`
- Provider-specific key-value pairs passed directly to the volume plugin (e.g., `type: gp3`, `iops: "3000"`, `encrypted: "true"` for AWS EBS).

---

## 5. Inspection, Verification & Troubleshooting Commands

```bash
# 1. List all StorageClasses (look for (default) annotation)
kubectl get sc
kubectl get storageclass

# 2. Inspect a specific StorageClass and its parameters
kubectl describe sc google-storage

# 3. View dynamically created PVs (Notice the Claim and StorageClass columns)
kubectl get pv

# 4. Check PVC status (should go from Pending -> Bound automatically)
kubectl get pvc
```