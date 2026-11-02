# IPAM & Pod CIDR in the CKA Exam: Common Scenarios and Simple Troubleshooting

In the **CKA (Certified Kubernetes Administrator)** exam, you will not get theoretical multiple-choice questions about IPAM. Instead, you will get hands-on tasks: inspecting cluster network configurations or troubleshooting nodes and pods stuck in `ContainerCreating` / `Pending` because of IP allocation issues.

This guide covers the key scenarios using **simple, step-by-step commands** that are easy to remember under exam pressure.

---

## Key Paths to Remember

Memorize these four locations. You can check them using simple `ls` and `cat`:

| Directory / File | What it Contains |
| :--- | :--- |
| `/etc/cni/net.d/` | CNI configuration files (defines IPAM type and subnet). |
| `/opt/cni/bin/` | CNI and IPAM binary executables (`bridge`, `host-local`, etc.). |
| `/var/lib/cni/networks/` | Host-local IP lease tracking files. |
| `/etc/kubernetes/manifests/kube-controller-manager.yaml` | Static pod manifest with `--cluster-cidr` and `--allocate-node-cidrs`. |

---

## Scenario 1: Inspecting the Configured IPAM Plugin & Subnet

### Exam Task
> *"Identify the CNI plugin and IPAM configuration used on `node01`."*

### Step 1: Switch to the node (if required)
```bash
ssh node01
```

### Step 2: List files in the CNI directory
```bash
ls /etc/cni/net.d/
```
*Example output:*
```text
10-flannel.conflist
```

### Step 3: View the file content
```bash
cat /etc/cni/net.d/10-flannel.conflist
```

Look for the `"ipam"` section inside the JSON:
```json
"ipam": {
  "type": "host-local",
  "subnet": "10.244.1.0/24"
}
```

* **IPAM Plugin:** `host-local`
* **Subnet:** `10.244.1.0/24`

---

## Scenario 2: Finding the Cluster-Wide Pod CIDR

### Exam Task
> *"What is the Pod Network CIDR configured for this cluster?"*

You can find this using simple commands from the control plane node.

### Option A: Check the Controller Manager Manifest
```bash
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cidr
```
*Expected output:*
```text
- --allocate-node-cidrs=true
- --cluster-cidr=10.244.0.0/16
```
The Pod CIDR is `10.244.0.0/16`.

### Option B: Check Kube-Proxy ConfigMap
```bash
kubectl get cm -n kube-system
kubectl describe cm kube-proxy -n kube-system | grep -i cidr
```

---

## Scenario 3: Pods Stuck in `ContainerCreating` (Node Missing `podCIDR`)

### Exam Task
> *"Pods scheduled on `node02` fail to start and remain in `ContainerCreating`. Identify and fix the issue."*

### Step 1: Describe the failing pod
```bash
kubectl get pods
kubectl describe pod <failing-pod-name>
```
Look at the **Events** section at the bottom. You will see an error like:
```text
Failed to create pod sandbox: plugin type="bridge" failed (add): failed to allocate for range: no IP addresses available
```

### Step 2: Check if the node has an assigned `podCIDR`
Run this simple `kubectl describe node` command:
```bash
kubectl describe node node02 | grep -i cidr
```

* **Healthy Node output:**
  ```text
  PodCIDR:                     10.244.2.0/24
  PodCIDRs:                    10.244.2.0/24
  ```
* **Broken Node output:**
  ```text
  PodCIDR:                     <none>
  ```

### Step 3: Why is PodCIDR missing?
If PodCIDR is `<none>`, the `kube-controller-manager` is not allocating CIDRs to nodes.

1. Open the static pod manifest on the control plane node:
   ```bash
   vi /etc/kubernetes/manifests/kube-controller-manager.yaml
   ```
2. Verify that both flags are present under `spec.containers.command`:
   ```yaml
   - --allocate-node-cidrs=true
   - --cluster-cidr=10.244.0.0/16
   ```
3. Save and exit (`:wq`).  
   Kubelet will automatically restart the `kube-controller-manager` pod.

---

## Scenario 4: Missing IPAM Binary (`host-local`)

### Exam Task
> *"A new node was added, but pods cannot get networking configured. Fix the node."*

### Step 1: Check Kubelet logs on the worker node
```bash
ssh node01
journalctl -u kubelet -e
```
*(Tip: `-e` jumps straight to the end of the logs).*

Look for error lines containing:
```text
failed to find plugin "host-local" in path [/opt/cni/bin]
```

### Step 2: Check the binary directory
```bash
ls /opt/cni/bin/
```
If `host-local` is missing from the list, the IPAM plugin executable was not installed or was moved.

### Step 3: Fix
Restore or copy the missing binary to `/opt/cni/bin/`:
```bash
cp /tmp/host-local /opt/cni/bin/
chmod +x /opt/cni/bin/host-local
```

---

## Scenario 5: Checking Active IP Allocations on a Node

### Exam Task
> *"Check which IPs are currently leased to pods on this node using the `host-local` IPAM plugin."*

Remember that `host-local` stores each assigned IP as a simple file on the node's filesystem.

### Step 1: List the network directory
```bash
ls /var/lib/cni/networks/
```
*Example output:*
```text
cni0
```

### Step 2: List the assigned IPs inside that network
```bash
ls /var/lib/cni/networks/cni0/
```
*Example output:*
```text
10.244.1.2  10.244.1.3  10.244.1.4  last_reserved_ip.0  lock
```
Every file named after an IP represents an active pod lease.

### Step 3: See which container owns an IP
```bash
cat /var/lib/cni/networks/cni0/10.244.1.2
```
This prints the Container ID that owns the IP `10.244.1.2`.

---

## Summary: Simple Command Workflow for CKA

```bash
# 1. Check CNI & IPAM configuration:
ls /etc/cni/net.d/
cat /etc/cni/net.d/<filename>

# 2. Check cluster-wide Pod CIDR:
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cidr

# 3. Check node's assigned Pod CIDR:
kubectl describe node <node-name> | grep -i cidr

# 4. Check available CNI/IPAM binaries:
ls /opt/cni/bin/

# 5. Check kubelet errors on a worker node:
journalctl -u kubelet -e
```

