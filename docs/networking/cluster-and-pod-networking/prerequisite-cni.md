# Container Network Interface (CNI): Standards, Architecture, and Mechanics

In earlier sections, we examined how to manually plumb container networks using Linux primitives: creating a `veth` pair, moving one end into a network namespace, attaching the peer to a bridge, assigning IP addresses, setting up default routes, and configuring `iptables` for NAT.

In the early container ecosystem, every runtime (Docker, rkt, Mesos, Kubernetes) implemented this plumbing independently using custom code. This resulted in duplicated effort, vendor lock-in, and an inability to swap networking implementations.

The **Container Network Interface (CNI)** was created to solve this problem by providing a minimal, language-agnostic specification that decouples **Container Runtimes** from **Network Providers**.

---

## 1. The Core Problem: The Need for a Standard

Consider two components:
1. **Container Runtimes:** Software responsible for running containers (Kubernetes/CRI-O, containerd, rkt).
2. **Network Solutions:** Technologies providing connectivity between containers (Linux Bridge, Flannel, Calico, Cilium, Weave Net).

Without a standard:
* If Kubernetes wanted to support 5 different network providers, Kubernetes developers had to write and maintain 5 separate internal networking drivers.
* If a vendor invented a new networking technology (e.g., eBPF-based Cilium), they had to submit patches to every container runtime in existence.

### The Solution: A Common Contract
CNI establishes an interface specification:
* The **Container Runtime** is solely responsible for creating the execution environment and network namespace.
* The **CNI Plugin** is an executable binary solely responsible for connecting that network namespace to the network fabric.

```text
┌──────────────────────────────────────────────┐
│        Container Runtime (e.g., containerd)  │
│  1. Creates network namespace                │
│  2. Reads CNI JSON configuration file        │
└──────────────────────┬───────────────────────┘
                       │ Invokes via ENV variables + STDIN
                       ▼
┌──────────────────────────────────────────────┐
│              CNI Plugin Binary               │
│         (e.g., bridge, calico, cilium)       │
│  1. Creates veth pair                        │
│  2. Moves interface into target netns        │
│  3. Invokes IPAM plugin to allocate IP       │
│  4. Configures routes, gateway, MTU          │
│  5. Returns JSON response to STDOUT          │
└──────────────────────────────────────────────┘
```

---

## 2. The CNI Specification: Roles & Responsibilities

The CNI specification strictly divides duties between the runtime and the plugin.

### Responsibilities of the Container Runtime
1. **Create the Network Namespace:** Before calling any CNI plugin, the runtime creates the target network namespace for the container/pod.
2. **Locate Network Configuration:** Finds the CNI configuration files (typically located in `/etc/cni/net.d/`).
3. **Execute the Plugin Binary:** Locates the plugin executable (typically in `/opt/cni/bin/`) and executes it as a standard OS process.
4. **Lifecycle Execution:**
   * Calls the plugin with command **`ADD`** when a container/pod is created.
   * Calls the plugin with command **`DEL`** when a container/pod is destroyed.
   * Calls the plugin with command **`CHECK`** to verify that the container's networking remains in the expected state.
5. **Clean up Namespaces:** Destroys the network namespace after the `DEL` command completes.

### Responsibilities of the CNI Plugin
1. **Implement Core Operations:** Must accept commands via the `CNI_COMMAND` environment variable:
   * **`ADD`:** Attach the container to the network.
   * **`DEL`:** Detach the container from the network and release resources.
   * **`CHECK`:** Probe container network health.
   * **`VERSION`:** Report supported CNI specification versions.
2. **IP Allocation:** Coordinate with an IPAM (IP Address Management) plugin to allocate and release IPs.
3. **Interface Plumbing:** Create the virtual links (`veth`, macvlan, etc.), configure link parameters (name, MTU, MAC address), and bring the link up inside the target namespace.
4. **Routing:** Configure the default gateway and any necessary static routes inside the namespace.
5. **Standard Output:** Return success results or structured error codes in a standardized JSON schema over `stdout`.

---

## 3. How Runtimes Communicate with CNI Plugins

CNI is **not a daemon, gRPC service, or library**. It is an **executable binary protocol**. The runtime executes the plugin as a subprocess, passing metadata through **Environment Variables** and **Standard Input (STDIN)**.

### Environment Variables Passed by Runtime
When invoking a plugin, the runtime sets several mandatory environment variables:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `CNI_COMMAND` | The action to execute (`ADD`, `DEL`, `CHECK`, `VERSION`). | `ADD` |
| `CNI_CONTAINERID` | Unique identifier for the container. | `a8b3c9f21d...` |
| `CNI_NETNS` | Absolute path to the network namespace file. | `/var/run/netns/red` (or `/proc/<pid>/ns/net`) |
| `CNI_IFNAME` | The interface name to create **inside** the container. | `eth0` |
| `CNI_ARGS` | Optional extra key-value pairs (e.g., K8s Pod Name, Namespace). | `K8S_POD_NAME=web;K8S_POD_NAMESPACE=prod` |
| `CNI_PATH` | Directories where dependent CNI plugin binaries reside. | `/opt/cni/bin` |

### Input via STDIN: The JSON Configuration
The runtime reads a JSON configuration file from disk (e.g., `/etc/cni/net.d/10-mynet.conf`) and pipes its content directly into the plugin's `stdin`:

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
    "subnet": "10.244.0.0/16",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

* **`type`:** Specifies which executable binary in `/opt/cni/bin/` to run (in this case, `/opt/cni/bin/bridge`).
* **`isGateway`:** Tells the bridge plugin to assign an IP to the host bridge interface (`cni0`) so it functions as a gateway.
* **`ipMasq`:** Configures `iptables` MASQUERADE rules so outbound packets can access the external LAN/Internet.
* **`ipam`:** Delegates IP allocation to a secondary plugin (here, `host-local`).

### Output via STDOUT: Structured JSON Response
Upon successful execution of `ADD`, the plugin writes a JSON payload to `stdout`:

```json
{
  "cniVersion": "1.0.0",
  "interfaces": [
    { "name": "cni0", "mac": "0a:58:0a:f4:00:01" },
    { "name": "vethb8d9c", "mac": "96:9a:28:e1:52:6a" },
    { "name": "eth0", "mac": "0a:58:0a:f4:00:02", "sandbox": "/var/run/netns/red" }
  ],
  "ips": [
    {
      "version": "4",
      "interface": 2,
      "address": "10.244.0.2/16",
      "gateway": "10.244.0.1"
    }
  ],
  "routes": [
    { "dst": "0.0.0.0/0", "gw": "10.244.0.1" }
  ]
}
```

---

## 4. Categories of CNI Plugins

The CNI project (maintained by the CNCF) provides a set of reference plugins stored in `/opt/cni/bin/`:

### A. Main Interface Plugins (Create the link)
* **`bridge`:** Creates a Linux bridge on the host and connects the container via a `veth` pair.
* **`vlan`:** Allocates a VLAN sub-interface on the host and moves it into the container.
* **`ipvlan`:** Uses Linux IPvlan drivers to share the host's MAC address while assigning unique IP addresses.
* **`macvlan`:** Assigns a unique MAC address directly to the container interface, making it look like a physical machine on the external switch.
* **`ptp`:** Creates a direct point-to-point `veth` pair between host and container without a bridge.

### B. IPAM Plugins (IP Address Management)
Network plugins delegate address tracking to IPAM plugins:
* **`host-local`:** Allocates IPs out of a locally defined subnet range and records leases in flat files on the host filesystem (`/var/lib/cni/networks/`).
* **`dhcp`:** Runs a local proxy daemon to lease IP addresses from an external network DHCP server.

### C. Meta / Chained Plugins
Plugins that modify or wrap an existing connection:
* **`portmap`:** An `iptables`-based plugin that handles port forwarding (hostPort mapping).
* **`tuning`:** Adjusts sysctl settings, MTU, or interface link parameters inside the container.
* **`bandwidth`:** Applies Linux Traffic Control (`tc`) rate-limiting (ingress/egress bandwidth shaping).

### D. Third-Party Kubernetes Network Providers
Enterprise Kubernetes networking solutions implement the CNI standard:
* **Flannel:** Simple Layer 3 overlay network (using VXLAN).
* **Calico:** High-performance, Layer 3 routing network using standard BGP (Border Gateway Protocol) and Linux kernel routing tables without overlays, plus granular Network Policies.
* **Cilium:** eBPF-based networking that bypasses traditional `iptables` and connection tracking directly in the kernel for high performance and security observability.
* **Weave Net, AWS VPC CNI, Azure CNI, etc.**

---

## 5. CNI vs. Docker's CNM (Container Network Model)

A common point of confusion is why Docker commands (`docker run`) do not directly accept standard CNI plugins like Calico or Flannel:

* **Docker did not adopt CNI.** Instead, Docker developed and champions its own standard called **CNM (Container Network Model)** via the `libnetwork` library.
* **Key Difference:**
  * **CNM (Docker):** Relies on a centralized daemon (`dockerd`) and defines abstractions around *Sandboxes*, *Endpoints*, and *Networks*.
  * **CNI:** Highly decentralized, minimalist UNIX-philosophy binary tool. It runs stateless executables via CLI/STDIN without a persistent networking daemon managing the interface creation lifecycle.

### How Kubernetes Uses Docker Despite the Difference
In older Kubernetes setups where Docker was the container engine, Kubernetes worked around this incompatibility:
1. When a Pod was scheduled, the `kubelet` invoked the Docker daemon to create a container with **no network configuration** (`--network=none`).
2. Docker created the container process and placed it in an isolated network namespace, but added no interfaces.
3. The `kubelet` then directly called the configured **CNI plugins** (e.g., `/opt/cni/bin/bridge` or `/opt/cni/bin/calico`), passing the path to that container's network namespace (`/proc/<pid>/ns/net`).
4. The CNI plugin performed all interface plumbing, IP assignment, and routing directly inside the namespace.

Modern Kubernetes clusters no longer use Docker; they use **CRI runtimes** (such as `containerd` or `CRI-O`) which support CNI natively out of the box.

---

## 6. Summary Comparison: Manual Plumbing vs. CNI

| Manual Step (Previous Lectures)      | CNI Equivalent                                         |
| :----------------------------------- | :----------------------------------------------------- |
| `ip netns add <name>`                | Handled automatically by the **Container Runtime**.    |
| `ip link add veth... type veth`      | Handled by the **Main CNI Plugin** (`bridge`, `ptp`).  |
| `ip link set dev ... master cni0`    | Configured via JSON directive: `"bridge": "cni0"`.     |
| Tracking allocated IP addresses      | Handled by the **IPAM Plugin** (`host-local`, `dhcp`). |
| `ip addr add <IP> dev eth0`          | Applied by CNI using details from IPAM.                |
| `ip route add default via ...`       | Declared in the `"routes"` block of the CNI config.    |
| `iptables -t nat -A POSTROUTING ...` | Enabled via `"ipMasq": true` in CNI config.            |