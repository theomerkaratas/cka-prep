# Static Pods

**Static Pods** are managed directly by the `kubelet` daemon on a specific node, without the `kube-apiserver` observing or managing them.

Unlike pods managed by the control plane (such as a Deployment or ReplicaSet), the `kubelet` directly watches the local files in a designated directory and restarts them if they crash.

---

## Key Characteristics of Static Pods

- **No Control Plane Dependency:** `kubelet` can create and run static pods even if the API server, etcd, or scheduler are down.
- **Pods Only:** `kubelet` only understands Pods. You **cannot** create Deployments, Services, or ReplicaSets as static resources.
- **Mirror Pods:** If an API server is running, the `kubelet` creates a **mirror pod** on the API server for each static pod. This allows them to be visible via `kubectl get pods`, but **they cannot be deleted or modified via `kubectl`**.
- **Deleting a Static Pod:** You must SSH into the node and remove or move the manifest file from the static pod directory.

---

## Locating the Static Pod Path

There are two primary ways `kubelet` determines the static pod path:

### Method 1: Check Kubelet Service Arguments
```bash
ps aux | grep kubelet
```
Look for `--pod-manifest-path` or `--config`:
```text
--config=/var/lib/kubelet/config.yaml
```

### Method 2: Inspect the Kubelet Configuration File
```bash
cat /var/lib/kubelet/config.yaml | grep -i staticpodpath
```
Output:
```yaml
staticPodPath: /etc/kubernetes/manifests
```

> In `kubeadm` setups, the control plane components (`kube-apiserver`, `etcd`, `kube-controller-manager`, `kube-scheduler`) are themselves deployed as static pods inside `/etc/kubernetes/manifests`.

---

## Creating a Static Pod (CKA Exam Workflow)

1. SSH into the designated node:
   ```bash
   ssh node01
   ```
2. Find the static pod directory from `/var/lib/kubelet/config.yaml`.
3. Create a pod manifest inside that directory:
   ```bash
   kubectl run static-busybox --image=busybox --dry-run=client -o yaml --command -- sleep 1000 > /etc/kubernetes/manifests/static-busybox.yaml
   ```
4. Return to the control plane and verify the mirror pod appears:
   ```bash
   kubectl get pods -A | grep static-busybox
   ```
   *(The pod name will be suffixed with the node name: `static-busybox-node01`)*.

---

## Static Pods vs. DaemonSets

| Feature | Static Pods | DaemonSets |
| :--- | :--- | :--- |
| **Manager** | Managed directly by the local `kubelet` | Managed by the `DaemonSetController` |
| **Lifecycle** | Created from local manifest files on disk | Created via Kubernetes API objects |
| **Scheduling** | Ignored by `kube-scheduler` | Scheduled by `kube-scheduler` (using NodeAffinity) |
| **Common Use** | Control plane components (`kube-apiserver`, `etcd`) | Node daemons (monitoring, logging, networking) |
| **Deletion** | Remove manifest file from `/etc/kubernetes/manifests` | `kubectl delete ds <name>` |
