# Kubernetes Pod Networking: Architecture, Manual Implementation, and CNI Automation

In earlier sections, we established how nodes communicate across a physical or virtual local area network (e.g., `192.168.1.0/24`), and how Linux namespaces, virtual ethernet pairs (`veth`), and software bridges provide isolated network stacks for containers.

In Kubernetes, worker and master nodes run hundreds or thousands of Pods. How these Pods receive IP addresses, communicate within the same node, and reach Pods across different nodes without Network Address Translation (NAT) constitutes the **Kubernetes Pod Network Model**.

---

## 1. The Kubernetes Networking Model: Core Requirements

Kubernetes does not provide an out-of-the-box native pod network implementation. Instead, it defines strict networking requirements that any underlying network solution (CNI plugin or manual routing) must satisfy:

1. **Unique IP per Pod:** Every Pod receives its own distinct, routable IP address. Containers within the same Pod share that IP address and port space (via a shared network namespace).
2. **Intra-Node Communication:** All Pods on a given node can communicate with all other Pods on that same node using their IP addresses, without NAT.
3. **Inter-Node Communication:** All Pods on any node can communicate with all Pods on all other nodes using their IP addresses, without NAT.
4. **Agent-to-Pod Communication:** Agents running on a node (such as the `kubelet` or system daemons) can communicate with all Pods running on that same node.
5. **No IP Masquerading (No NAT):** The IP that a Pod sees as its own address must be identical to the IP that every other Pod or node in the cluster sees when receiving traffic from it.

---

## 2. Cluster Topology Setup

To understand how to implement this model from scratch using Linux primitives, consider a three-node cluster:

```text
 Physical / Host Network: 192.168.1.0/24
        ┌──────────────────────────────────┬───────────────────────────────┐
        │                                  │                               │
        ▼                                  ▼                               ▼
 ┌───────────────┐                  ┌───────────────┐               ┌───────────────┐
 │    Node 1     │                  │    Node 2     │               │    Node 3     │
 │ 192.168.1.11  │                  │ 192.168.1.12  │               │ 192.168.1.13  │
 └───────────────┘                  └───────────────┘               └───────────────┘
```

To prevent IP collisions across nodes, the overall cluster Pod network (Cluster CIDR, e.g., `10.244.0.0/16`) is divided into distinct, non-overlapping subnets allocated per node:

* **Node 1 Subnet:** `10.244.1.0/24` (Hosts Pods `10.244.1.2` through `10.244.1.254`)
* **Node 2 Subnet:** `10.244.2.0/24` (Hosts Pods `10.244.2.2` through `10.244.2.254`)
* **Node 3 Subnet:** `10.244.3.0/24` (Hosts Pods `10.244.3.2` through `10.244.3.254`)

---

## 3. Phase 1: Configuring Node-Level Pod Bridges

Each node requires a local Layer 2 virtual switch (a Linux bridge) to which that node's Pods will attach.

### Step 1: Create the Bridge Interface
On each node, create a bridge named `v-net-0` (or `cni0` in production environments):

```bash
# Executed on Node 1, Node 2, and Node 3:
ip link add v-net-0 type bridge
```

### Step 2: Bring the Bridge Link UP
```bash
# Executed on Node 1, Node 2, and Node 3:
ip link set dev v-net-0 up
```

### Step 3: Assign the Gateway IP to the Bridge
The bridge acts as the default gateway for Pods residing on that node. Each node assigns an IP from its allocated pod subnet to its local bridge:

```bash
# On Node 1:
ip addr add 10.244.1.1/24 dev v-net-0

# On Node 2:
ip addr add 10.244.2.1/24 dev v-net-0

# On Node 3:
ip addr add 10.244.3.1/24 dev v-net-0
```

---

## 4. Phase 2: Attaching Pods to the Bridge (Same-Node Communication)

When a Pod is scheduled on Node 1, a dedicated network namespace is instantiated for it. Connecting this namespace to `v-net-0` requires four sequential steps:

```text
                              NODE 1 (192.168.1.11)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                        v-net-0 (10.244.1.1/24)                              │
│                        [Layer 2 Virtual Switch]                             │
│                               ▲         ▲                                   │
│               veth-pod1-br    │         │    veth-pod2-br                   │
│                               ▼         ▼                                   │
│                         veth-pod1     veth-pod2                             │
│                      ┌────────────┐ ┌────────────┐                          │
│                      │   Pod 1    │ │   Pod 2    │                          │
│                      │ 10.244.1.2 │ │ 10.244.1.3 │                          │
│                      └────────────┘ └────────────┘                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Step 1: Create the Virtual Cable (`veth` pair)
Create a point-to-point link with two endpoints:
```bash
ip link add veth-pod1 type veth peer name veth-pod1-br
```

### Step 2: Attach the Peer to the Bridge
Bind the host end of the pair (`veth-pod1-br`) to the bridge `v-net-0` and bring it up:
```bash
ip link set veth-pod1-br master v-net-0
ip link set dev veth-pod1-br up
```

### Step 3: Move the Other End into the Pod's Namespace
```bash
ip link set veth-pod1 netns <pod1-namespace>
```

### Step 4: Configure IP Address and Default Gateway Inside the Pod
Inside the Pod's namespace, assign its unique IP address from the node's subnet, bring the interface up, and set the bridge as the default gateway:
```bash
# Assign IP address
ip -n <pod1-namespace> addr add 10.244.1.2/24 dev veth-pod1

# Bring interface up
ip -n <pod1-namespace> link set dev veth-pod1 up

# Add default route via the node's bridge IP
ip -n <pod1-namespace> route add default via 10.244.1.1
```

### How Same-Node Traffic Flows
When Pod 1 (`10.244.1.2`) communicates with Pod 2 (`10.244.1.3`) on Node 1:
1. Pod 1 inspects its subnet mask (`/24`) and identifies that `10.244.1.3` is on the local subnet.
2. Pod 1 issues an ARP request over `veth-pod1`.
3. The bridge `v-net-0` switches the ARP frame to `veth-pod2-br`.
4. Pod 2 responds with its MAC address.
5. All subsequent Layer 2 frames switch directly across `v-net-0` without reaching Node 1's physical routing layer or traversing NAT.

---

## 5. Phase 3: Cross-Node Pod Communication (Inter-Node Routing)

Consider what happens when Pod 1 on Node 1 (`10.244.1.2`) sends a packet to Pod 3 on Node 2 (`10.244.2.2`).

```text
 Pod 1 (10.244.1.2)                                          Pod 3 (10.244.2.2)
         │                                                           ▲
         ▼                                                           │
 v-net-0 (10.244.1.1)                                        v-net-0 (10.244.2.1)
         │                                                           ▲
         ▼                                                           │
 Node 1 (192.168.1.11) ──(Physical LAN / Gateway)──► Node 2 (192.168.1.12)
```

### The Breakdown Without Routing
1. Pod 1 checks the destination IP `10.244.2.2`. It is outside its local `/24` subnet.
2. Pod 1 sends the packet to its default gateway: `10.244.1.1` (`v-net-0`).
3. The packet enters Node 1's root network namespace.
4. Node 1 inspects its kernel routing table for `10.244.2.2`.
5. **The Failure:** Node 1 has no route for `10.244.2.0/24`. It either discards the packet or pushes it out to its own default internet gateway, which drops the private IP.

### Solution A: Host-Level Static Routing
To satisfy the Kubernetes requirement of direct, non-NAT pod-to-pod routing, every node's kernel routing table must know which physical node hosts which Pod subnet.

On **Node 1**, add routes to Node 2 and Node 3:
```bash
ip route add 10.244.2.0/24 via 192.168.1.12
ip route add 10.244.3.0/24 via 192.168.1.13
```

On **Node 2**, add routes to Node 1 and Node 3:
```bash
ip route add 10.244.1.0/24 via 192.168.1.11
ip route add 10.244.3.0/24 via 192.168.1.13
```

On **Node 3**, add routes to Node 1 and Node 2:
```bash
ip route add 10.244.1.0/24 via 192.168.1.11
ip route add 10.244.2.0/24 via 192.168.1.12
```

### End-to-End Packet Walkthrough
1. **Source:** `10.244.1.2` -> **Destination:** `10.244.2.2`.
2. Pod 1 routes the packet to `v-net-0` (`10.244.1.1`).
3. Node 1 matches the route: `10.244.2.0/24 via 192.168.1.12`.
4. Node 1 places the packet onto the physical wire (`eth0`) destined for Node 2's MAC address. **The IP headers remain unchanged** (Source: `10.244.1.2`, Destination: `10.244.2.2`).
5. Node 2 receives the packet on its physical interface.
6. Because `net.ipv4.ip_forward = 1` is enabled on Node 2, the kernel inspects the destination IP (`10.244.2.2`), recognizes it belongs to its local `v-net-0` bridge (`10.244.2.0/24`), and forwards it across the bridge into Pod 3's namespace.
7. Return traffic follows the identical path in reverse via Node 2's static route back to `192.168.1.11`.

### Solution B: Physical / Upstream Router Table
Managing static routes on every individual host scales poorly as nodes are added or removed ($O(N^2)$ routes across $N$ nodes).

A scalable alternative is configuring a central core router on the underlying network with the static routes:
```text
Router Routing Table:
- 10.244.1.0/24 -> Gateway: 192.168.1.11
- 10.244.2.0/24 -> Gateway: 192.168.1.12
- 10.244.3.0/24 -> Gateway: 192.168.1.13
```
Each node simply points its default gateway to the central router, which routes traffic between the node subnets. This aggregates individual node subnets into a unified cluster-wide CIDR: `10.244.0.0/16`.

---

## 6. Phase 4: Automating with CNI Scripts

In a cluster creating and destroying hundreds of Pods dynamically, running manual `ip` commands is not viable. These operations are packaged into an automated script adhering to the **CNI specification**.

### The Script Architecture (`net-script.sh`)

A CNI plugin must support two primary operations passed as arguments: **`ADD`** and **`DEL`**.

```bash
#!/usr/bin/env bash
# net-script.sh - Minimal CNI-compliant script

COMMAND=$1
CONTAINER_ID=$2
NETNS=$3

case "$COMMAND" in
  add)
    # 1. Create veth pair
    VETH="veth-${CONTAINER_ID:0:5}"
    VETH_BR="${VETH}-br"
    ip link add "$VETH" type veth peer name "$VETH_BR"

    # 2. Attach host end to bridge
    ip link set "$VETH_BR" master v-net-0
    ip link set dev "$VETH_BR" up

    # 3. Move pod end to container network namespace
    ip link set "$VETH" netns "$NETNS"

    # 4. Allocate and assign IP (retrieved from an IPAM store)
    POD_IP="10.244.1.2/24"
    ip -n "$NETNS" addr add "$POD_IP" dev "$VETH"
    ip -n "$NETNS" link set dev "$VETH" up

    # 5. Set default gateway to node bridge IP
    ip -n "$NETNS" route add default via 10.244.1.1
    ;;

  del)
    # 1. Identify interface from container and delete
    # Deleting one end of a veth pair automatically destroys the peer
    VETH="veth-${CONTAINER_ID:0:5}"
    ip -n "$NETNS" link del "$VETH"
    ;;

  *)
    echo "Unknown command: $COMMAND"
    exit 1
    ;;
esac
```

---

## 7. How the Container Runtime Invokes CNI

The container runtime (such as `containerd` or `CRI-O`) managed by the `kubelet` drives this workflow automatically.

```text
 ┌────────────────────────────────────────────────────────┐
 │                   Container Runtime                    │
 │                                                        │
 │  Flags configured at startup:                          │
 │    --cni-conf-dir = /etc/cni/net.d                     │
 │    --cni-bin-dir  = /opt/cni/bin (or /etc/cni/bin)     │
 └──────────────────────────┬─────────────────────────────┘
                            │
                            ▼
 1. Reads configuration from /etc/cni/net.d/10-net-script.conf
 2. Locates binary in /opt/cni/bin/net-script.sh
 3. Executes:
    ./net-script.sh add <container-id> <namespace-path>
```

### The Execution Sequence When a Pod Is Scheduled:
1. The **`kubelet`** receives a pod manifest from the API server and instructs the **Container Runtime** to create a Pod sandbox.
2. The runtime creates a new Linux network namespace for the Pod.
3. The runtime reads the first alphabetically ordered configuration file in `--cni-conf-dir` (`/etc/cni/net.d/`).
4. The runtime identifies the plugin type specified in the JSON file and locates the matching executable in `--cni-bin-dir` (`/opt/cni/bin/`).
5. The runtime executes the plugin binary with arguments `add <container_id> <netns_path>`.
6. The plugin creates the `veth` pair, wires one end to the node bridge, assigns an IP address from IPAM, configures the default route, and returns a JSON payload to the runtime.
7. The runtime starts the application containers inside the prepared network namespace.

---

## 8. Summary Comparison: Manual Steps vs. CNI Abstraction

| Purpose | Manual Shell Commands | Production CNI Plugin (Flannel, Calico, Cilium) |
| :--- | :--- | :--- |
| **Node Switch** | `ip link add v-net-0 type bridge` | Automated bridge (`cni0`) or eBPF virtual datapath. |
| **Node Gateway** | `ip addr add 10.244.x.1/24 dev v-net-0` | Automatically configured based on `spec.podCIDR` of the Node object. |
| **Pod Plumbing** | `ip link add veth... type veth` | Executed dynamically inside the CNI binary on `ADD`. |
| **IP Tracking** | Manual selection or flat file storage | Delegated to IPAM plugins (`host-local`, Calico IPAM, or etcd). |
| **Inter-Node Routing**| `ip route add 10.244.y.0/24 via <Node-IP>` | Managed via VXLAN/Geneve encapsulation (Flannel) or BGP route peering (Calico). |