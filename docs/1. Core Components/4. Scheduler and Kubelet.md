# Scheduler and Kubelet

Understanding the separation of concerns between `kube-scheduler` and `kubelet` is crucial for CKA.

---

## 1. Kube-Scheduler

`kube-scheduler` is responsible for deciding **which node** should run a pod. It does **not** actually start the containers on that node—that is the job of the `kubelet`.

### How Scheduling Works
1. **Filter Nodes:** Filters out nodes that do not meet the pod's requirements (e.g., insufficient CPU/RAM, taints without matching tolerations, node selectors).
2. **Score Nodes:** Ranks the remaining nodes on a scale (0–100) based on resource utilization, affinity rules, etc.
3. **Bind:** Selects the highest-scoring node, updates the pod's `spec.nodeName` field, and writes the binding to `kube-apiserver`.

### Deployment
- In `kubeadm` clusters, it runs as a static pod: `/etc/kubernetes/manifests/kube-scheduler.yaml`.
- In manual installations, it runs as a systemd service: `/etc/systemd/system/kube-scheduler.service`.

---

## 2. Kubelet

The `kubelet` is the primary agent that runs on **every node** (both control plane and worker nodes). It is often called the "captain of the node".

> **Important:** Unlike other Kubernetes components, the `kubelet` is **never deployed as a container or Pod**. It always runs as a native system daemon managed by `systemd`.

### Primary Responsibilities
- Registers the worker node with the Kubernetes cluster.
- Receives `PodSpec` definitions from the `kube-apiserver` (or local manifest files).
- Communicates with the container runtime (e.g., `containerd` via Container Runtime Interface - CRI) to create, start, and stop containers.
- Monitors container and pod health (executes liveness, readiness, and startup probes) and reports status back to `kube-apiserver`.

### Kubelet Configuration and Static Pods

Kubelet service configuration typically points to a configuration file:

```ini
# /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
ExecStart=/usr/local/bin/kubelet \
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
  --config=/var/lib/kubelet/config.yaml \
  --kubeconfig=/etc/kubernetes/kubelet.conf \
  --v=2
```

The static pod directory is configured inside `/var/lib/kubelet/config.yaml`:

```yaml
# /var/lib/kubelet/config.yaml
staticPodPath: /etc/kubernetes/manifests
```

### Inspecting and Troubleshooting Kubelet

```bash
# Check service status
systemctl status kubelet

# View live kubelet logs
journalctl -u kubelet -f

# Check running process and flags
ps aux | grep kubelet

# Verify static pod directory
grep -i staticpod /var/lib/kubelet/config.yaml
```


