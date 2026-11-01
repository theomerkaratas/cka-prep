# Concepts, Deployment Mechanics, and CKA Exam Strategy
When bootstrapping a Kubernetes cluster from scratch using `kubeadm`, the cluster initially appears operational, but running `kubectl get nodes` shows the nodes in a **`NotReady`** state, and the internal DNS pods (`coredns`) remain stuck in **`Pending`**.

This occurs because Kubernetes **does not provide a built-in network implementation**. Kubernetes defines the **Networking Model specification**, but delegates its implementation to external **Network Addons (CNI Plugins)**.

Below is a technical guide explaining what addons are, how they are deployed, how to configure them, and how to handle them during the **CKA (Certified Kubernetes Administrator)** exam.

---

## 1. What is a Network Addon?

In Kubernetes terminology, an **Addon** is a cluster feature implemented through standard Kubernetes objects (typically `DaemonSets`, `Deployments`, `ConfigMaps`, and `RBAC` roles) running inside the cluster (usually in the `kube-system` namespace).

A **Network Addon** is a package that deploys a specific CNI provider (such as Calico, Flannel, Cilium, or historically Weave Net) onto the cluster.

### What the Network Addon Does When Deployed:
1. **Runs as a `DaemonSet`:** Ensures that one agent pod runs on every single node (master and workers) in the cluster.
2. **Installs CNI Binaries:** The addon pod mounts the host's `/opt/cni/bin/` directory and writes its CNI executable binaries there (e.g., `calico`, `flannel`, `loopback`).
3. **Writes the CNI Configuration:** It mounts the host's `/etc/cni/net.d/` directory and writes the JSON configuration file (e.g., `10-calico.conflist`).
4. **Triggers `kubelet` to Transition to `Ready`:**
   * The `kubelet` continuously inspects `/etc/cni/net.d/` for a valid network configuration.
   * As soon as the addon writes this file, `kubelet` initializes its network provider, changes the node condition from `NetworkPluginNotReady` to `Ready`, and schedules `coredns` pods.
5. **Establishes the Pod Overlay/Routing Fabric:** The addon's daemon establishes routing tables, VXLAN tunnels, or BGP peering between nodes so Pods on Node A can communicate directly with Pods on Node B.

---

## 2. Why the Official Documentation is Vendor-Neutral

The note highlights a deliberate architectural and editorial policy of the Kubernetes project:

* **Vendor Neutrality:** The Kubernetes project (under the CNCF) avoids endorsing or favoring any specific commercial or open-source networking vendor.
* **Documentation Boundaries:** The official Kubernetes documentation ([kubernetes.io/docs](https://kubernetes.io/docs/)) lists approved addons and links to their project pages, but **does not host third-party installation manifests** or vendor-specific command snippets (e.g., `kubectl apply -f https://raw.githubusercontent.com/...`).
* **External Links are Out-of-Scope:** In the official CKA exam environment, internet access is strictly restricted. You can access `kubernetes.io` and its subdomains, but **you cannot follow external redirects** to GitHub, Calico's documentation, or other third-party vendor websites.

---

## 3. How Network Addons are Handled in the CKA Exam

Candidates often worry: *"If the docs don't contain the manifest URL and I can't browse GitHub, how do I install a CNI plugin in the exam?"*

The exam design accounts for this explicitly:

### Rule 1: Manifests or Commands are Provided
If a task requires you to initialize a cluster with `kubeadm` and install a network addon, the exam task text **will provide**:
* The direct command to execute, OR
* A local file path containing the YAML manifest (e.g., `/opt/manifests/calico.yaml`), OR
* An internal, allowed link to apply (e.g., `kubectl apply -f <provided-internal-url>`).

You are **never** expected to memorize long GitHub manifest URLs or guess vendor-specific YAML structures.

### Rule 2: Alignment with `--pod-network-cidr`
When deploying a network plugin, its IP range **must match** the Pod CIDR passed during `kubeadm init`.

* Example: If you initialize the cluster with:
  ```bash
  kubeadm init --pod-network-cidr=10.244.0.0/16
  ```
* Flannel defaults to `10.244.0.0/16`. It works out of the box.
* Calico defaults to `192.168.0.0/16`. If using Calico with the above CIDR, you must edit the Calico manifest to set `CALICO_IPV4POOL_CIDR` to `10.244.0.0/16` before applying, or initialize `kubeadm` with `192.168.0.0/16`.

---

## 4. Practical Mechanics: How an Addon is Applied and Verified

### Applying a Network Addon
Typically, applying an addon requires a single `kubectl` command against the manifest:

```bash
# Applying a provided manifest file or URL
kubectl apply -f /path/to/cni-addon.yaml
```

### Verification Checklist (Troubleshooting Steps)
Whenever an addon is applied, verify the following four steps to ensure cluster networking is functional:

#### 1. Check Addon Pods in `kube-system`
The network plugin pods must reach the `Running` state on all nodes:
```bash
kubectl get pods -n kube-system -o wide
```
Look for pods prefixed with `calico-node-*`, `kube-flannel-ds-*`, or `cilium-*`.

#### 2. Verify Node Status Transitions to `Ready`
Once the CNI plugin initializes:
```bash
kubectl get nodes
```
All nodes should transition from `NotReady` to **`Ready`**.

#### 3. Verify `CoreDNS` Starts
Prior to CNI installation, `coredns` pods remain in `Pending`. Once CNI is active:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```
Both `coredns` pods must show `Running`.

#### 4. Verify Host CNI Configuration
On any node, inspect the local filesystem to confirm the plugin has written its configuration:
```bash
# Check CNI binaries installed on the host:
ls -la /opt/cni/bin/

# Check the active CNI configuration file:
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/*conflist
```

---

## 5. Summary Cheat Sheet for the CKA Exam

| Concept                      | Key Takeaway                                                                                         |
| :--------------------------- | :--------------------------------------------------------------------------------------------------- |
| **Why Nodes are `NotReady`** | No CNI plugin installed; `kubelet` is waiting for `/etc/cni/net.d/` config.                          |
| **Why CoreDNS is `Pending`** | CoreDNS requires Pod-to-Pod network routing, which does not exist until CNI is applied.              |
| **Documentation Limit**      | External links on `kubernetes.io` to GitHub or vendor docs are blocked in the exam.                  |
| **Exam Guarantee**           | All required CNI manifest paths, URLs, or explicit commands are provided in the question prompt.     |
| **CIDR Matching**            | Always ensure the CNI IP pool configuration matches the `--pod-network-cidr` used in `kubeadm init`. |
| **Key Host Directories**     | Config: `/etc/cni/net.d/` \| Binaries: `/opt/cni/bin/`                                               |