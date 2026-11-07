# IP Address Management (IPAM) in Kubernetes: Mechanics, CNI Plugins, and Subnet Allocation

When a Pod is scheduled onto a node, it receives its own unique IP address from a private cluster IP range. However, **who** determines which IP range belongs to which node? **Who** assigns the IP to the Pod's network namespace? And **how** does the system ensure that two Pods never get the same IP address (preventing IP collisions)?

This process is known as **IP Address Management (IPAM)**.

> [!NOTE]
> IPAM in this context does **not** manage the physical or virtual IPs assigned to the Kubernetes nodes themselves (e.g., `eth0` on worker/master nodes). Node IPs are managed externally by your cloud provider, network administrator, or a local DHCP server.  
> **IPAM here specifically governs:**
> 1. How virtual bridge networks on each node obtain their Pod CIDR / Subnet.
> 2. How individual Pods obtain unique IP addresses within that subnet.
> 3. How allocated IP addresses are tracked and released when Pods are terminated.

---

## 1. The Core Architectural Question: Who is Responsible?

According to the official Kubernetes and CNI network model:
* **Kubernetes itself does not assign Pod IPs.** Kubernetes specifies only the *contract*: every Pod must have a unique IP, and all Pods must be able to communicate with all other Pods without NAT.
* **The CNI Plugin is responsible.** CNI (Container Network Interface) dictates that the network provider (or custom script) must handle IP allocation when plumbing the container namespace.

```text
┌────────────────────────────────────────────────────────┐
│                      kubelet                           │
│  "Create a Pod, here is the network namespace"         │
└───────────────────────────┬────────────────────────────┘
                            │ Invokes CNI (ADD)
                            ▼
┌────────────────────────────────────────────────────────┐
│                   Main CNI Plugin                      │
│               (e.g., bridge, calico)                   │
│                                                        │
│   1. Plumbs veth pair & bridge                         │
│   2. Calls IPAM Plugin ───► ┌───────────────────────┐  │
│                             │      IPAM Plugin      │  │
│                             │ (host-local / dhcp)   │  │
│                             │                       │  │
│                             │ Allocates unique IP:  │  │
│                             │ 10.244.1.2/24         │  │
│                             └───────────┬───────────┘  │
│   3. Configures IP on eth0 ◄────────────┘              │
│   4. Sets Default Gateway & Routes                     │
└────────────────────────────────────────────────────────┘
```

Rather than building complex IP tracking logic into every main network plugin (like `bridge` or `ptp`), the CNI specification decouples IP allocation into dedicated **IPAM Plugins**.

---

## 2. Managing IPs: From Naive Scripting to Built-in Plugins

### The Naive Approach: Flat File on the Host
If you were writing a custom network script from scratch:
1. You would define a subnet for the node (e.g., `10.244.1.0/24`).
2. When creating a container, you need to track which IPs are free and which are taken.
3. The simplest mechanism is a text file on the host filesystem (e.g., `/etc/cni/net.d/ips.txt`):
   ```text
   10.244.1.2
   10.244.1.3
   ```
4. Your script would read this file, pick the next available unused IP, assign it to the container interface (`eth0`), append the allocated IP to the file, and place file locks to prevent race conditions when multiple containers spin up concurrently.
5. On container deletion, your script would remove that IP from the file.

### The CNI Solution: Off-the-Shelf IPAM Plugins
Instead of writing and maintaining lock-safe IP tracking logic yourself, CNI provides standard, pre-compiled IPAM plugin binaries (residing in `/opt/cni/bin/`):

| IPAM Plugin | Location | How It Works | Best Used For |
| :--- | :--- | :--- | :--- |
| **`host-local`** | `/opt/cni/bin/host-local` | Allocates IPs strictly from a local subnet defined on that host. Tracks allocated leases as flat files in `/var/lib/cni/networks/<network-name>/`. | Default for bridge-based clusters, Flannel, standard Kubeadm setups. |
| **`dhcp`** | `/opt/cni/bin/dhcp` | Runs a background daemon (`dhcp daemon`) that queries an external physical DHCP server on the local LAN to lease an IP for each Pod. | Environments where containers must be first-class citizens on the corporate network/VLAN. |
| **Provider-Specific** | e.g. `calico-ipam` | Stores allocations in a distributed datastore (`etcd` or K8s CRDs) and dynamically assigns IP blocks (CIDR blocks) to nodes on demand. | Enterprise overlays and large-scale clusters. |

---

## 3. How `host-local` IPAM Works in Practice

The most commonly tested and used IPAM plugin in standard Linux/CKA environments is **`host-local`**.

### Where is it Configured?
The IPAM configuration is nested inside the CNI network configuration JSON file located in `/etc/cni/net.d/`:

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
    "rangeStart": "10.244.1.10",
    "rangeEnd": "10.244.1.200",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ],
    "dataDir": "/var/lib/cni/networks"
  }
}
```

### Key Directives in the `ipam` Block:
* **`type`**: Specifies which binary in `/opt/cni/bin/` to invoke for IP management (`host-local`).
* **`subnet`**: The IP range dedicated to this specific node's Pods (`10.244.1.0/24`).
* **`rangeStart` / `rangeEnd`** *(Optional)*: Limits allocations to a specific slice of the subnet, reserving remaining addresses for gateways or static services.
* **`routes`**: Default gateway and static routing rules pushed into the Pod's network namespace. `{ "dst": "0.0.0.0/0" }` directs all outbound traffic through the bridge gateway (`cni0`).
* **`dataDir`**: The directory on the host where the IP lease state files are stored (default: `/var/lib/cni/networks/`).

---

## 4. State Persistence: How Leases are Stored on Disk

Unlike complex database solutions, `host-local` uses the Linux filesystem as its atomic database:

```bash
# Inspect the active IP allocations on the host:
ls -la /var/lib/cni/networks/mynet/
```

Inside this directory, each allocated IP is recorded as an individual file named after that IP address:
```text
/var/lib/cni/networks/mynet/
├── 10.244.1.2
├── 10.244.1.3
├── 10.244.1.4
├── last_reserved_ip.0
└── lock
```

* **Content of each IP file:** The container ID that currently holds the lease (e.g. `9f8b2c4e...`).
* **`last_reserved_ip.0`:** A pointer to the last allocated IP to ensure round-robin sequential allocation.
* **`lock`:** A filesystem mutex lock ensuring safe concurrent execution when multiple Pods start at the exact same millisecond.

### Lifecycle Events:
1. **Pod Creation (`ADD`):**
   * The container runtime calls the main CNI plugin (e.g., `bridge`).
   * The `bridge` plugin calls `/opt/cni/bin/host-local` with `CNI_COMMAND=ADD`.
   * `host-local` checks `/var/lib/cni/networks/mynet/`, finds the next free IP (e.g. `10.244.1.5`), writes the container ID into `/var/lib/cni/networks/mynet/10.244.1.5`, and outputs the allocated IP to `stdout` as JSON.
   * The `bridge` plugin assigns `10.244.1.5/24` to `eth0` inside the Pod namespace.
2. **Pod Deletion (`DEL`):**
   * The container runtime invokes CNI with `CNI_COMMAND=DEL`.
   * The `bridge` plugin calls `/opt/cni/bin/host-local` with `CNI_COMMAND=DEL`.
   * `host-local` deletes `/var/lib/cni/networks/mynet/10.244.1.5`.
   * That IP is immediately returned to the pool for reuse.

---

## 5. How Nodes Get Their Node-Specific Subnets (Pod CIDR)

A frequent source of confusion: *If `host-local` runs locally on each node, how do we make sure Node 1 and Node 2 don't use the exact same subnet (e.g., both using `10.244.1.0/24`)?*

In a multi-node Kubernetes cluster:
* The cluster has a global **Pod Network CIDR** (e.g. `10.244.0.0/16`).
* Each node is assigned a non-overlapping slice (typically a `/24`, providing 254 IPs per node):
  * **Node 01:** `10.244.1.0/24`
  * **Node 02:** `10.244.2.0/24`
  * **Node 03:** `10.244.3.0/24`

### How this assignment happens:
1. **Kubernetes Controller Manager (`NodeIPAMController`):**
   When `kube-controller-manager` is started with `--allocate-node-cidrs=true` and `--cluster-cidr=10.244.0.0/16`, it automatically carves up the cluster CIDR and assigns a unique `spec.podCIDR` to every node object:
   ```bash
   kubectl get node node01 -o jsonpath='{.spec.podCIDR}'
   # Output: 10.244.1.0/24
   ```
2. **CNI DaemonSet / Agent:**
   The CNI agent running on each node (such as Flannel or Calico) reads this `spec.podCIDR` from the Node object and writes the local configuration file into `/etc/cni/net.d/` with the appropriate node subnet.

---

## 6. Summary Comparison: IPAM Mechanisms

| Approach | IP Storage Mechanism | Collision Avoidance | Scope |
| :--- | :--- | :--- | :--- |
| **Custom Flat File** | Custom script & text files | Custom file locks | Single host only |
| **`host-local` (CNI)** | Flat files in `/var/lib/cni/networks/` | Host-level atomic file locks | Single host (relies on distinct per-node subnets) |
| **`dhcp` (CNI)** | External DHCP Server | DHCP protocol leases & ARP/ICMP probes | Entire physical/virtual L2 broadcast domain |
| **`calico-ipam`** | `etcd` / Kubernetes CRDs (`IPPools`, `BlockAffinity`) | Distributed transactions / optimistic locking | Entire multi-node Kubernetes cluster |

