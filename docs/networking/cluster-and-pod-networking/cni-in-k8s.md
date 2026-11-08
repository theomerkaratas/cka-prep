# CNI in Kubernetes: Runtime Integration, Configuration Architecture, and Lifecycle Execution

In the previous sections, we established the core mechanics of network namespaces, virtual ethernet (`veth`) pairs, Linux software bridges, and the CNI specification. 

This section examines how **Kubernetes actually implements and executes CNI**: which components trigger network setup, how configuration files are structured, how binaries are discovered, and how to troubleshoot CNI-related cluster failures in production and on the **CKA (Certified Kubernetes Administrator)** exam.

---

## 1. Architectural Overview: Who Invokes CNI in Kubernetes?

A common misconception is that the Kubernetes control plane (e.g., `kube-apiserver` or `kube-scheduler`) configures pod networking. In reality, network plumbing happens entirely on the local worker node.

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                              WORKER NODE                               │
 │                                                                        │
 │  ┌─────────────────┐        CRI (gRPC)         ┌────────────────────┐  │
 │  │     kubelet     ├──────────────────────────►│ Container Runtime  │  │
 │  │                 │  RunPodSandbox Request    │(containerd / CRI-O)│  │
 │  └─────────────────┘                           └─────────┬──────────┘  │
 │                                                          │             │
 │                                     1. Reads config from │             │
 │                                        /etc/cni/net.d/   │             │
 │                                     2. Invokes binary in │             │
 │                                        /opt/cni/bin/     │             │
 │                                                          ▼             │
 │                                                ┌────────────────────┐  │
 │                                                │ CNI Plugin Binary  │  │
 │                                                │(bridge, calico,..) │  │
 │                                                └─────────┬──────────┘  │
 │                                                          │             │
 │                                                          ▼             │
 │                                                ┌────────────────────┐  │
 │                                                │ Pod Net Namespace  │  │
 │                                                │  (eth0, IP, Route) │  │
 │                                                └────────────────────┘  │
 └────────────────────────────────────────────────────────────────────────┘
```

### The Chain of Invocation
1. **`kubelet`:** Watches the Kubernetes API server. When a Pod is scheduled to its node, the `kubelet` makes a **Container Runtime Interface (CRI)** gRPC call (`RunPodSandbox`) to the local container runtime.
2. **Container Runtime (`containerd`, `CRI-O`):** 
   * Responsible for creating the container processes and isolating them within Linux namespaces.
   * Creates the network namespace for the Pod sandbox (the `pause` container).
   * **Directly invokes the CNI plugin** to configure network connectivity for that newly created namespace before starting the application containers.
3. **CNI Plugin (`/opt/cni/bin/*`):** Executes as an ephemeral binary process, plumbs the network interfaces, allocates an IP address, sets default routes, and exits, returning status to the runtime.

---

## 2. Standard Filesystem Locations

By convention and standard specification, the container runtime expects CNI components to reside in two specific directories on every node:

```text
/etc/cni/net.d/   ──► Stores network configuration files (JSON / .conflist)
/opt/cni/bin/     ──► Stores compiled executable plugin binaries
```

### A. The Plugin Binary Directory: `/opt/cni/bin/`
Contains the stateless executable binaries. When inspecting this directory on a functional Kubernetes node:

```bash
ls -l /opt/cni/bin/
```

Common binaries found here include:
* **Interface creation:** `bridge`, `ipvlan`, `macvlan`, `ptp`, `host-device`, `vlan`
* **IP Address Management (IPAM):** `host-local`, `dhcp`, `static`
* **Meta/Chaining plugins:** `portmap` (for `hostPort`), `tuning` (for sysctl/MTU), `bandwidth`, `firewall`
* **Third-party orchestrators:** `calico`, `calico-ipam`, `flannel`, `cilium-cni`

### B. The Configuration Directory: `/etc/cni/net.d/`
Contains JSON configuration files defining the active network topology.

```bash
ls -l /etc/cni/net.d/
```

Example output:
```text
-rw-r--r-- 1 root root 384 Nov  1 10:00 10-mynet.conflist
-rw-r--r-- 1 root root 220 Nov  1 10:00 99-loopback.conf
```

### The Lexicographical Ordering Rule
A node may have multiple configuration files in `/etc/cni/net.d/`. 

The CNI specification dictates that **the container runtime must pick the first file in alphabetical (lexicographical) order**.

* Because `10-mynet.conflist` sorts before `99-loopback.conf`, the runtime selects `10-mynet.conflist` as the primary network specification.
* This is why CNI configuration filenames are almost always prefixed with numbers (`10-`, `20-`, `50-`) to enforce deterministic precedence.

---

## 3. Dissecting a CNI Configuration File

Below is a standard CNI bridge configuration (`/etc/cni/net.d/10-bridge.conf`) mapped directly to the Linux networking primitives discussed in earlier chapters:

```json
{
  "cniVersion": "1.0.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.1.0/24",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

### Detailed Field Breakdown and Kernel Actions

| Configuration Key | Value | Underlying Kernel Action / Purpose |
| :--- | :--- | :--- |
| **`cniVersion`** | `"1.0.0"` | Declares the CNI specification version the schema adheres to. |
| **`name`** | `"mynet"` | A logical identifier for the network. |
| **`type`** | `"bridge"` | **The binary to execute.** The runtime searches `/opt/cni/bin/` for an executable named `bridge`. |
| **`bridge`** | `"cni0"` | The name of the Linux software bridge to create or attach to on the host (`ip link add cni0 type bridge`). |
| **`isGateway`** | `true` | Tells the plugin to assign an IP address to the `cni0` bridge interface itself (`ip addr add 10.244.1.1/24 dev cni0`) so that the host can act as the **default gateway** for all local pods. |
| **`ipMasq`** | `true` | Configures an `iptables` / `nftables` Source NAT rule: `iptables -t nat -A POSTROUTING -s 10.244.1.0/24 ! -o cni0 -j MASQUERADE`. This allows pods to reach external networks using the node's physical IP. |
| **`ipam.type`** | `"host-local"` | **Delegated IPAM plugin.** The bridge plugin calls `/opt/cni/bin/host-local` to allocate an IP address from the defined range. |
| **`ipam.subnet`** | `"10.244.1.0/24"`| The unique Pod CIDR assigned to this specific node. |
| **`ipam.routes`** | `[{ "dst": "0.0.0.0/0" }]` | Injects a default route inside the Pod's network namespace pointing to the bridge IP (`ip route add default via 10.244.1.1`). |

---

## 4. IPAM Plugins: `host-local` vs. `dhcp`

Managing IP assignment across dynamic containers requires dedicated IP Address Management (IPAM) plugins.

```text
               ┌──────────────────────────────────────────────┐
               │              IPAM Strategies                 │
               └──────────────────────┬───────────────────────┘
                                      │
            ┌─────────────────────────┴─────────────────────────┐
            ▼                                                   ▼
┌───────────────────────────────┐               ┌───────────────────────────────┐
│          host-local           │               │             dhcp              │
├───────────────────────────────┤               ├───────────────────────────────┤
│ • Zero network overhead       │               │ • Queries external network    │
│ • State kept in flat files    │               │   DHCP daemon                 │
│ • Path: /var/lib/cni/networks/│               │ • Requires running dhcp-daemon│
│ • Default for K8s subnets     │               │ • Used for direct LAN bridges │
└───────────────────────────────┘               └───────────────────────────────┘
```

### 1. `host-local` (Standard Kubernetes Strategy)
* **Mechanics:** Each node is assigned a dedicated slice of the cluster subnet (e.g., Node 1 gets `10.244.1.0/24`, Node 2 gets `10.244.2.0/24`). The `host-local` binary runs locally on the node and manages IP allocation independently without network consensus.
* **Storage Location:** Allocations are written to local disk files under `/var/lib/cni/networks/<network-name>/`:
  ```bash
  ls /var/lib/cni/networks/mynet/
  # Output shows reserved IPs:
  # 10.244.1.1  10.244.1.2  10.244.1.3  last_reserved_ip.0
  ```
  Each file contains the container ID holding that IP lease. Deleting a Pod removes the corresponding file.

### 2. `dhcp`
* **Mechanics:** Useful when containers must receive IP addresses from an external corporate network router.
* **Requirement:** The host must run a persistent background daemon (`/opt/cni/bin/dhcp daemon`) that proxies DHCP requests between the container interfaces and the external DHCP server.

---

## 5. CNI Plugin Chaining (`.conflist`)

In modern Kubernetes clusters, a Pod often needs more than just a bridge and an IP. It may require:
1. Basic Layer 2 connectivity (`bridge`).
2. Port mapping support (`portmap`).
3. Network policy enforcement (`firewall`).
4. Bandwidth limits (`bandwidth`).

To execute multiple plugins sequentially on a single Pod, CNI supports **Network Configuration Lists (`.conflist`)**:

```json
{
  "cniVersion": "1.0.0",
  "name": "k8s-pod-network",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipam": {
        "type": "host-local",
        "subnet": "10.244.1.0/24"
      }
    },
    {
      "type": "portmap",
      "capabilities": { "portMappings": true }
    },
    {
      "type": "bandwidth"
    }
  ]
}
```

* When a Pod starts, the runtime executes the plugins in top-to-bottom array order (`bridge` -> `portmap` -> `bandwidth`).
* When the Pod terminates, the runtime calls the same plugins in reverse order (`bandwidth` -> `portmap` -> `bridge`).

---

## 6. CKA Troubleshooting: Diagnosing CNI Failures

Understanding CNI configuration is essential for diagnosing node and pod creation issues on the CKA exam.

### Problem Scenario 1: Node is in `NotReady` Status
When a new worker node joins a cluster, `kubectl get nodes` frequently shows:

```text
NAME     STATUS     ROLES    AGE   VERSION
node01   NotReady   <none>   30s   v1.30.0
```

#### Why?
The `kubelet` monitors `/etc/cni/net.d/`. If the directory is empty, missing, or contains unreadable/corrupted configuration files, the `kubelet` sets the condition `NetworkPluginNotReady` and refuses to mark the node `Ready`.

#### Diagnostic Command:
```bash
kubectl describe node node01 | grep -i network
# Output reveals:
# KubeletNotReady: container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```

#### Fix:
Verify that `/etc/cni/net.d/` contains a valid configuration file and that `/opt/cni/bin/` contains the corresponding binaries. Once a CNI provider (such as Calico or Flannel) is applied, the config file is placed into `/etc/cni/net.d/`, and the node transitions to `Ready`.

---

### Problem Scenario 2: Pods Stuck in `ContainerCreating`
When a Pod is scheduled to a node, but fails to start:

```bash
kubectl get pods
# Output:
# nginx-pod   0/1   ContainerCreating   0   2m
```

#### Root Cause Investigation:
Run `kubectl describe pod <pod-name>` and inspect the **Events** section at the bottom:

```text
Events:
  Type     Reason                  Age                From               Message
  ----     ------                  ----               ----               -------
  Warning  FailedCreatePodSandBox  45s (x3 over 2m)   kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox: failed to find plugin "bridge" in path [/opt/cni/bin]
```

* **Root Cause:** The configuration `/etc/cni/net.d/10-bridge.conf` requested `"type": "bridge"`, but the binary `/opt/cni/bin/bridge` is missing or lacks executable permissions (`chmod +x`).

---

## 7. Diagnostic and Verification Commands Summary

| Task / Objective | Command |
| :--- | :--- |
| **Check installed CNI binaries** | `ls -l /opt/cni/bin/` |
| **Check active CNI configurations** | `ls -l /etc/cni/net.d/` |
| **Inspect primary CNI configuration** | `cat /etc/cni/net.d/*.conf* \| head -n 30` |
| **Inspect local IPAM allocated leases**| `ls -l /var/lib/cni/networks/*` |
| **Inspect host bridge created by CNI**| `ip addr show cni0` (or `v-net-0`) |
| **Check Kubelet network registration** | `systemctl status kubelet -l` |
| **Check Node CNI status** | `kubectl describe node <name> \| grep NetworkPlugin` |
| **Check Pod CNI sandbox failure logs** | `kubectl describe pod <name>` (inspect Events) |