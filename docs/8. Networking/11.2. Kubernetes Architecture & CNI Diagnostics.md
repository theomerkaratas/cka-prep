## 1. Demystifying `Kubelet` Configuration & The CRI Socket

### The Investigative Chain: How to Locate `Kubelet` Parameters

In Kubernetes, file paths and startup arguments can vary across operating systems, distributions, and versions. Rather than guessing paths, query the process supervisor: **`systemd`**.

```bash
systemctl cat kubelet
```

This output reveals two critical sections:

1. **The Base Unit (`kubelet.service`):** Defines the base execution command (`ExecStart=/usr/bin/kubelet`), but lacks arguments.
2. **The Kubeadm Drop-in (`10-kubeadm.conf`):** Defines runtime environment variables:
    - `Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"`: Identifies the primary configuration file.
    - `EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env`: Identifies dynamically generated runtime flags.
    - `ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS`: Shows how the variables assemble into the execution command.

### Identifying the Container Runtime Endpoint

To find the CRI (Container Runtime Interface) endpoint that `kubelet` uses:

1. **Option A (Inspect `config.yaml`):**
    
    ```bash
    grep -i "containerRuntimeEndpoint" /var/lib/kubelet/config.yaml
    ```
    
    _Expected Output:_
    
    ```yaml
    containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
    ```
    
2. **Option B (Inspect `kubeadm-flags.env`):**
    
    ```bash
    cat /var/lib/kubelet/kubeadm-flags.env
    ```
    
3. **Option C (Query the live process):**
    
    ```bash
    ps -ef | grep kubelet | grep -o -- '--container-runtime-endpoint=[^ ]*'
    ```
    

---

## 2. What Is `containerRuntimeEndpoint` and Why Does It Use a `.sock` File?

### Architecture & Separation of Concerns

- The `kubelet` is an orchestrator agent on the node; it does not pull images or manage container isolation directly.
- The **Container Runtime** (`containerd`, `CRI-O`) performs container lifecycle operations.
- Communication between two distinct local processes on a Linux system is handled via a **UNIX Domain Socket (`.sock`)**, which provides low-latency, bidirectional inter-process communication (IPC) managed by the kernel without network stack overhead.

```text
┌─────────────┐                                   ┌──────────────┐
│   kubelet   │ ──── (gRPC over UNIX Socket) ────►│  containerd  │
└─────────────┘   unix:///var/run/.../containerd.sock └──────────────┘
```

### Additional Configuration Parameters in `/var/lib/kubelet/config.yaml`

Inspecting this file reveals other critical node configurations:

- **`staticPodPath: /etc/kubernetes/manifests`:** Directory watched by `kubelet` to run control-plane static Pods.
- **`cgroupDriver: systemd`:** The cgroup driver used for container resource tracking.
- **`clusterDNS: [172.20.0.10]`:** The IP address of the in-cluster CoreDNS service.

---

## 3. CNI Filesystem Hierarchy: `/opt/cni/bin` vs. `/etc/cni/net.d`

In the Kubernetes CNI standard and the Linux Filesystem Hierarchy Standard (FHS):

|Directory|Purpose|Types of Files Contained|Examples|
|---|---|---|---|
|**`/opt/cni/bin`**|**Executables / Binaries**|Compiled programs and shell scripts that configure interfaces.|`bridge`, `flannel`, `calico`, `host-local`, `loopback`|
|**`/etc/cni/net.d`**|**Configurations**|Declarative configuration files consumed by the runtime.|`10-bridge.conf`, `10-calico.conflist`|

### Why Do Third-Party Binaries Live in `/opt`?

- **`/opt` (Optional):** Reserved under the Linux Filesystem Hierarchy Standard (FHS) for self-contained, third-party software packages and add-ons that are not part of the base operating system.
- **Isolation from Package Managers:** Core OS commands managed by `apt` or `dnf` live in `/usr/bin`. Placing CNI plugins into `/opt/cni/bin` prevents naming collisions and dependency corruption during operating system upgrades.
- **Clean Deletion:** Removing `/opt/cni` removes all plugin binaries without leaving orphaned files scattered across system paths.

### Quick Mental Map

- **Looking for configurations/settings?** ➡️ Look in **`/etc`** (`/etc/cni/net.d/`)
- **Looking for executable binaries/plugins?** ➡️ Look in **`/opt`** (`/opt/cni/bin/`)
- **Looking for dynamically allocated state/leases?** ➡️ Look in **`/var`** (`/var/lib/cni/networks/`)

---

## 4. Practical CKA Troubleshooting: Finding Installed vs. Configured CNI Plugins

Questions in exams and troubleshooting scenarios often ask two distinct questions:

### Question Type A: "Which plugin is NOT available / installed on this host?"

- **Concept:** This asks about the installed executable binaries available on the filesystem.
- **Verification Command:**
    
    ```bash
    ls /opt/cni/bin
    ```
    
- **Analysis:**
    - The directory contains plugins like `bridge`, `dhcp`, `flannel`, `host-local`, `ipvlan`, `loopback`, `macvlan`, `portmap`, `ptp`, `tuning`, `vlan`.
    - If a choice mentions a vendor name that has no binary in `/opt/cni/bin` (e.g., `cisco`), it is **not available**.

---

### Question Type B: "What is the CNI plugin CONFIGURED to be used on this cluster?"

- **Concept:** This asks which plugin is actively selected and evaluated by the container runtime for pod networking.

#### Method 1: Inspect Node CNI Configuration Directory (Primary Method)

The container runtime reads configuration files from `/etc/cni/net.d/`:

```bash
ls -l /etc/cni/net.d/
```

Inspect the active configuration (or the first file in alphabetical order):

```bash
cat /etc/cni/net.d/* | grep -E '"type"|"name"'
# or view the full configuration:
cat /etc/cni/net.d/*
```

- `"type": "flannel"` ➡️ **Flannel** is configured.
- `"type": "calico"` ➡️ **Calico** is configured.
- `"type": "weave-net"` ➡️ **Weave** is configured.
- `"type": "bridge"` ➡️ **Bridge** is configured.

#### Method 2: Inspect Active Network DaemonSets/Pods via `kubectl`

Third-party CNI providers run as DaemonSets in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system
```

- `kube-flannel-ds-...` ➡️ **Flannel**
- `calico-node-...` or `canal-...` ➡️ **Calico**
- `weave-net-...` ➡️ **Weave**
- `cilium-...` ➡️ **Cilium**

---

### Key Takeaway for CKA

|Question Phrasing|Where to Look|Command|
|---|---|---|
|**Available / Installed binaries?**|Node binary directory|`ls /opt/cni/bin`|
|**Configured / Active CNI plugin?**|Node config directory or cluster pods|`cat /etc/cni/net.d/*`  <br>`kubectl get pods -n kube-system`|