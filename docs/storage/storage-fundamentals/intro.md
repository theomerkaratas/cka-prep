# Storage in Kubernetes: Introduction

Managing data storage is a critical aspect of Kubernetes administration. Container filesystems are ephemeral by default—when a container crashes or is restarted by the `kubelet`, all data written to its local filesystem is permanently lost.

To provide statefulness, durability, and data sharing, Kubernetes decouples storage from individual container lifecycles.

---

## The Two Foundational Container Storage Concepts

Before diving into Kubernetes-native primitives, it is essential to understand the two core mechanisms inherited from Docker and container engines:

1. **Storage Drivers (Ephemeral Filesystem Layer):**
   - Manage the copy-on-write (CoW) layered union filesystem of images and running containers.
   - Examples: `overlay2`, `btrfs`, `zfs`.
   - Data stored in this layer is **ephemeral** and destroyed when the container is deleted.

2. **Volume Drivers / Plugins (Persistent Storage):**
   - Decouple data from the ephemeral container layer and mount storage from the host or external storage systems (e.g. AWS EBS, NFS, Ceph, iSCSI).
   - Data stored in volumes **persists** across container restarts and pod reschedule events.

---

## Roadmap of Storage Topics

In this section, we cover the full Kubernetes storage architecture required for the CKA exam:
- **Docker Storage Drivers & Volumes:** Layering, Copy-on-Write, and volume vs. bind mounts.
- **CRI, CNI, and CSI Interfaces:** How the Container Storage Interface (CSI) standardizes storage plugins.
- **Kubernetes Volumes:** Pod-level storage (`emptyDir`, `hostPath`, cloud disks).
- **PersistentVolumes (PV):** Cluster-scoped storage resource pools managed by administrators.
- **PersistentVolumeClaims (PVC):** Namespace-scoped storage requests made by developers.
- **PV Lifecycle & Reclaim Policies:** `Retain`, `Delete`, and `Recycle`.
- **StorageClasses:** Enabling automated, dynamic volume provisioning on demand.
