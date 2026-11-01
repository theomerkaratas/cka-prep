The **Container Storage Interface (CSI)** is an industry-standard specification enabling third-party storage vendors to write plugins that provision, attach, and mount storage across any container orchestration platform (Kubernetes, Mesos, Cloud Foundry) without modifying the core orchestrator codebase.

---

## 1. Architectural Evolution: Why Kubernetes Uses Standard Interfaces

In early Kubernetes versions, support for container runtimes, networking plugins, and storage providers was baked directly into the Kubernetes core source code (**in-tree**). This created high maintenance overhead, tight coupling, and slowed release cycles.

To decouple orchestrator logic from vendor-specific implementations, three standard interfaces were established:

```
               +-------------------------------------------+
               |        Kubernetes Control Plane           |
               +-------------------------------------------+
                   |                 |              |
                   v                 v              v
             +-----------+     +-----------+  +-----------+
             |    CRI    |     |    CNI    |  |    CSI    |
             +-----------+     +-----------+  +-----------+
                   |                 |              |
         [Container Runtimes]    [Networking]  [Storage Providers]
          - containerd            - Calico      - AWS EBS
          - CRI-O                 - Cilium      - Azure Disk
          - Docker(cri-dockerd).  - Flannel     - Portworx / Ceph
```

### The Three Standard Interfaces Compared

| Interface | Full Name | Standard Scope | Function in Kubernetes |
| :--- | :--- | :--- | :--- |
| **CRI** | Container Runtime Interface | Kubernetes-specific standard | Defines communication between `kubelet` and container runtimes (e.g., `containerd`, `CRI-O`). |
| **CNI** | Container Network Interface | Universal (CNCF project) | Defines how network connectivity, IP assignment, and interfaces are configured for pods. |
| **CSI** | Container Storage Interface | Universal industry standard | Defines how storage volumes are provisioned, attached, mounted, and deleted across orchestrators. |

> ⚠️ **Important Architectural Note:**
> **CSI is NOT Kubernetes-specific.** It is an open, vendor-neutral specification adopted by Kubernetes, Cloud Foundry, Mesos, and Nomad. Any storage plugin conforming to CSI can work with any CSI-compliant orchestrator out of the box.

---

## 2. How CSI Works: The RPC Model (Remote Procedure Calls)

The CSI specification defines a standard gRPC contract. Kubernetes (via helper sidecars and `kubelet`) issues standardized Remote Procedure Calls (**RPCs**), and the storage vendor's driver implements the actual execution logic:

```text
Kubernetes Event             Standard CSI RPC                  Storage Backend
+--------------------+       +----------------------+          +----------------------+
| Pod requests       | ----> | CreateVolume()       | -------> | Storage array        |
| PVC storage        |       +----------------------+          | allocates LUN        |
+--------------------+                                         +----------------------+
          |                                                               |
          v                                                               v
+--------------------+       +----------------------+          +----------------------+
| Pod scheduled      | ----> | Controller           | -------> | Attach block         |
| to Node            |       | PublishVolume()      |          | device to VM         |
+--------------------+       +----------------------+          +----------------------+
          |                                                               |
          v                                                               v
+--------------------+       +----------------------+          +----------------------+
| Pod starting on    | ----> | NodeStage() &        | -------> | Format ext4/xfs      |
| Node filesystem    |       | NodePublish()        |          | & mount to pod       |
+--------------------+       +----------------------+          +----------------------+
          |                                                               |
          v                                                               v
+--------------------+       +----------------------+          +----------------------+
| PVC deleted        | ----> | DeleteVolume()       | -------> | Decommission &       |
| by user            |       +----------------------+          | destroy storage      |
+--------------------+                                         +----------------------+
```

### Primary CSI RPC Operations:
- **`CreateVolume`**: Asks the storage driver to dynamically provision a new volume of a specified size and parameters on the storage array.
- **`ControllerPublishVolume`**: Attaches the created volume to the host VM/node where the pod will run.
- **`NodeStageVolume` / `NodePublishVolume`**: Formats the filesystem (e.g., `ext4`, `xfs`) and mounts the volume into the pod's container directory path.
- **`DeleteVolume`**: Decommissions, cleans up, and deletes the volume from the storage array when the resource is deleted.

---

## 3. Kubernetes In-Tree vs. Out-of-Tree (CSI) Storage

| Characteristic | In-Tree Volume Plugins (Deprecated) | Out-of-Tree CSI Drivers (Modern) |
| :--- | :--- | :--- |
| **Location of Code** | Inside `kubernetes/kubernetes` repository | Standalone repository maintained by storage vendor |
| **Release Cycles** | Tied directly to Kubernetes release cadence | Vendors release updates and fixes independently |
| **Cluster Privilege** | Runs inside `kube-controller-manager` with high privilege | Runs as standard Pods/DaemonSets via CSI sidecars |
| **Status in CKA** | Deprecated / migrated to CSI equivalents | Standard mechanism tested in CKA |

---

## 4. Inspection & Verification Commands (CKA Exam)

In CKA troubleshooting scenarios, you often need to verify if the CSI storage drivers and plugins are registered and healthy on cluster nodes:

```bash
# Check registered CSIDrivers in the cluster
kubectl get csidrivers
kubectl describe csidriver <driver-name>

# Check CSI node information (which drivers run on which node)
kubectl get csinodes
kubectl describe csinode <node-name>

# Check storage capacity reported by CSI drivers (if supported)
kubectl get csistoragecapacities -A

# Check driver pods (usually DaemonSets/Deployments in kube-system or storage namespace)
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
kubectl get daemonsets -A | grep -i csi

# Inspect StorageClasses configured to use CSI provisioners
kubectl get sc
kubectl describe sc <storage-class-name>
```

> `lsblk` is cool command