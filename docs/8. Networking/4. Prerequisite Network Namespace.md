# Linux Network Namespaces: Isolation, Virtual Ethernet Pairs, and Bridge Routing

Containers (Docker, `containerd`, Kubernetes Pods) are not lightweight virtual machines; they are standard Linux processes isolated by Linux kernel features: **`cgroups`** (resource limits) and **namespaces** (visibility boundaries).

Among these, the **Network Namespace (`netns`)** provides complete network stack virtualization. Below is a step-by-step breakdown of how network namespaces operate, how virtual cables and virtual switches work, and how traffic is routed between isolated namespaces, the host, and external networks.

---

## 1. What is a Namespace?

In Linux, a namespace restricts what a process can **see**:

* **PID Namespace:** A process inside a container sees itself as Process ID 1 (`PID 1`). On the underlying host, the exact same process runs with a regular, non-1 PID (such as `PID 4821`). The host sees all processes; the container sees only its own process subtree.
* **Network Namespace (`netns`):** A logical copy of the entire network stack. A process assigned to a network namespace has its own private:
  * Network interfaces (virtual or physical)
  * Routing tables
  * ARP tables
  * Firewall rules (`iptables` / `nftables`)
  * Socket listing (`ss`, `netstat`)

By default, an unmodified Linux system runs all processes inside a single **root network namespace**.

---

## 2. Creating and Inspecting Network Namespaces

Network namespaces are managed using the `ip netns` utility.

### Step 1: Create Two Namespaces
Create two isolated namespaces named `red` and `blue`:

```bash
ip netns add red
ip netns add blue
```

List all created namespaces:
```bash
ip netns
# Output:
# blue
# red
```

At this stage, Linux creates entry points for these namespaces in `/var/run/netns/`.

### Step 2: Executing Commands Inside a Namespace
To run a command inside an isolated namespace, use `ip netns exec <namespace> <command>`:

```bash
# Check interfaces on the root host:
ip link
# Displays: lo, eth0

# Check interfaces inside the "red" namespace:
ip netns exec red ip link
# Displays ONLY: lo (and it is in DOWN state)
```

Alternatively, the `-n` flag can be passed directly to the `ip` tool:
```bash
ip -n red link
```

### Observation: Complete Isolation
1. **Interfaces:** The `eth0` interface belonging to the host is completely invisible inside `red`.
2. **ARP Table:** Running `arp -n` on the host displays active neighbors. Running `ip -n red neigh` returns an empty table.
3. **Routing Table:** Running `ip -n red route` returns nothing.

The `red` and `blue` namespaces are completely isolated network silos with zero connectivity.

---

## 3. Connecting Two Namespaces: Virtual Ethernet Pairs (`veth`)

To connect two physical machines, an Ethernet cable is plugged into network cards on both ends.

To connect two network namespaces on the same host, Linux provides a **Virtual Ethernet Pair (`veth`)**. A `veth` pair acts as a bidirectional virtual wire: packets sent into one end automatically emerge from the other end.

```text
       ┌───────────────┐               ┌───────────────┐
       │ Namespace red │               │ Namespace blue│
       │               │   veth pair   │               │
       │   [veth-red]  │◄═════════════►│  [veth-blue]  │
       │ 192.168.15.1  │               │ 192.168.15.2  │
       └───────────────┘               └───────────────┘
```

### Step 1: Create the Virtual Cable
Create a `veth` pair where one end is named `veth-red` and the other is `veth-blue`:

```bash
ip link add veth-red type veth peer name veth-blue
```
At this point, both interfaces exist in the host's root namespace.

### Step 2: Attach Each End to its Respective Namespace
Move each interface into its target namespace:

```bash
ip link set veth-red netns red
ip link set veth-blue netns blue
```

If you run `ip link` on the host now, `veth-red` and `veth-blue` are gone; they have been moved into the isolated stacks.

### Step 3: Assign IP Addresses and Bring Interfaces UP
Assign IP addresses within the same subnet (`192.168.15.0/24`):

```bash
# Configure the red namespace:
ip -n red addr add 192.168.15.1/24 dev veth-red
ip -n red link set veth-red up

# Configure the blue namespace:
ip -n blue addr add 192.168.15.2/24 dev veth-blue
ip -n blue link set veth-blue up
```

*(Note: Always include the subnet mask `/24`. Without it, the kernel defaults to `/32`, which prevents local subnet route generation).*

### Step 4: Verify Communication and ARP Resolution
From the `red` namespace, ping `blue`:

```bash
ip netns exec red ping -c 2 192.168.15.2
```

Inspect the ARP table inside `red`:
```bash
ip -n red neigh
# Output:
# 192.168.15.2 dev veth-red lladdr <MAC_OF_VETH_BLUE> REACHABLE
```

* `red` learned `blue`'s MAC address through ARP over the virtual wire.
* The host's ARP table remains completely unaware of this transaction because the traffic never crossed the root namespace.

---

## 4. Connecting Multiple Namespaces: The Virtual Switch (Linux Bridge)

A point-to-point `veth` pair works only for two namespaces. If a third namespace (`green`) is added, direct cabling is unscalable (requiring an $N \times (N-1) / 2$ mesh of links).

To connect three or more namespaces, a **Virtual Switch** is required. In Linux, this is a **Bridge interface**.

```text
                     ┌──────────────────┐
                     │    HOST (Root)   │
                     │  vnet0 (Bridge)  │
                     └─▲──────▲───────▲─┘
                       │      │       │
            ┌──────────┘      │       └──────────┐
  veth-red-br                 veth-blue-br       veth-green-br
       ▲                      ▲                  ▲
       │ (veth pair)          │ (veth pair)      │ (veth pair)
       ▼                      ▼                  ▼
  veth-red               veth-blue          veth-green
┌──────────────┐       ┌──────────────┐   ┌──────────────┐
│Namespace red │       │Namespace blue│   │Namespace grn │
│192.168.15.1  │       │192.168.15.2  │   │192.168.15.3  │
└──────────────┘       └──────────────┘   └──────────────┘
```

### Step 1: Clean Up the Old Direct Cable
Deleting one end of a `veth` pair automatically destroys the peer interface:

```bash
ip -n red link delete veth-red
```

### Step 2: Create and Bring Up the Linux Bridge
Create a bridge named `vnet0` on the host:

```bash
ip link add vnet0 type bridge
ip link set dev vnet0 up
```

### Step 3: Create Cables for Each Namespace and Attach to Bridge
For each namespace, create a `veth` pair:
* One end connects inside the namespace.
* The other end connects to the host bridge (`vnet0`).

```bash
# --- Connect RED ---
ip link add veth-red type veth peer name veth-red-br
ip link set veth-red netns red
ip link set veth-red-br master vnet0

ip -n red addr add 192.168.15.1/24 dev veth-red
ip -n red link set veth-red up
ip link set veth-red-br up

# --- Connect BLUE ---
ip link add veth-blue type veth peer name veth-blue-br
ip link set veth-blue netns blue
ip link set veth-blue-br master vnet0

ip -n blue addr add 192.168.15.2/24 dev veth-blue
ip -n blue link set veth-blue up
ip link set veth-blue-br up
```

The `master vnet0` command binds the host end of the `veth` pair to the bridge, turning the interface into a switch port.

Now `red` (`192.168.15.1`) and `blue` (`192.168.15.2`) can communicate with each other through the Layer 2 bridge `vnet0`.

---

## 5. Connecting the Host to the Namespaces

At this stage, namespaces can talk to each other, but the **host itself cannot ping `192.168.15.1`**.

### Why?
The host sees `vnet0` as a network interface, but `vnet0` has **no IP address assigned to it**. The host does not know what local subnet `vnet0` belongs to.

### The Bridge Dual Identity:
* For the namespaces, `vnet0` is a **Layer 2 Switch**.
* For the host, `vnet0` is a **Layer 3 Network Interface**.

To enable host-to-namespace communication, assign an IP address from the `192.168.15.0/24` subnet directly to `vnet0`:

```bash
ip addr add 192.168.15.5/24 dev vnet0
```

Now the host has an interface on the `192.168.15.0/24` network. The host can ping `192.168.15.1` (`red`), and `red` can ping `192.168.15.5` (the host).

---

## 6. Connecting Namespaces to External Networks (Egress Routing & NAT)

Assume the host has a physical interface `eth0` with IP `192.168.1.2`, connected to a physical LAN where another server exists at `192.168.1.3`.

If namespace `blue` attempts to ping `192.168.1.3`:
```bash
ip netns exec blue ping 192.168.1.3
# Output:
connect: Network is unreachable
```

### Cause 1: Missing Default Gateway inside Namespace
`blue`'s routing table only contains a route for its local subnet (`192.168.15.0/24`). It has no route for `192.168.1.0/24`.

Add a default route inside `blue` pointing to the host's bridge interface (`192.168.15.5`):

```bash
ip -n blue route add default via 192.168.15.5
```

Now `blue` forwards external packets to the host.

---

### Cause 2: IP Forwarding on the Host
If you ping now, the error changes from `Network is unreachable` to silent packet loss (timeouts).

As established in basic routing mechanics, the Linux host drops transit packets between `vnet0` and `eth0` unless IP forwarding is explicitly enabled:

```bash
sysctl -w net.ipv4.ip_forward=1
```

---

### Cause 3: Asymmetric Routing / Missing Return Path (The Need for NAT)
With IP forwarding enabled, packets reach the external host (`192.168.1.3`). 
* Source IP: `192.168.15.2`
* Destination IP: `192.168.1.3`

When `192.168.1.3` sends a reply back to `192.168.15.2`, **it drops the packet**. The external physical LAN has no routing entry telling it that the `192.168.15.0/24` subnet lives behind `192.168.1.2`.

### The Solution: Network Address Translation (NAT / Masquerade)
The host must masquerade outbound packets from the private namespace subnet so they appear to originate from the host's own physical IP (`192.168.1.2`):

```bash
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -o eth0 -j MASQUERADE
```

* **How it operates:** When an outbound packet leaves `eth0`, `iptables` rewrites the Source IP from `192.168.15.2` to `192.168.1.2`.
* The external target (`192.168.1.3`) replies directly to `192.168.1.2`.
* The host receives the reply, consults its connection tracking table (`conntrack`), translates the Destination IP back to `192.168.15.2`, and routes it across `vnet0` back into the namespace.

Now namespaces have full outbound internet/LAN access.

---

## 7. Exposing Namespace Services to the Outside (Ingress / Port Forwarding)

What if a web server running inside the `blue` namespace (`192.168.15.2:80`) must be accessible to external clients on the LAN?

External devices cannot directly route to `192.168.15.2`. Traffic must enter via the host's physical IP (`192.168.1.2`) on a designated port (e.g., port `8080`) and be forwarded to the namespace.

This is configured using **Destination NAT (DNAT)** in `iptables`:

```bash
iptables -t nat -A PREROUTING -d 192.168.1.2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.15.2:80
```

* Any traffic arriving at the host on port `8080` has its destination rewritten to `192.168.15.2:80`.
* The kernel routes the packet through `vnet0` into `blue`.
* This mechanism is exactly how `docker run -p 8080:80` and Kubernetes `NodePort` services publish container ports to external networks.

---

## 8. Summary of Components & Commands

| Component / Action | Linux Command | Purpose |
| :--- | :--- | :--- |
| **Create namespace** | `ip netns add <name>` | Instantiates an isolated network stack. |
| **Execute in namespace**| `ip -n <name> <cmd>` | Runs network commands in namespace context. |
| **Create virtual wire** | `ip link add <v1> type veth peer name <v2>` | Creates a point-to-point Layer 2 virtual pair. |
| **Attach link to netns**| `ip link set <v1> netns <name>` | Moves one end of the pair into the namespace. |
| **Create virtual switch**| `ip link add <name> type bridge` | Creates a Layer 2 virtual Ethernet bridge. |
| **Attach port to bridge**| `ip link set <iface> master <bridge>` | Plugs a virtual interface into the bridge. |
| **Set namespace default GW**| `ip -n <ns> route add default via <IP>` | Directs outbound traffic to the bridge interface. |
| **Outbound NAT (Egress)**| `iptables -t nat -A POSTROUTING -s <CIDR> -o eth0 -j MASQUERADE` | Masks namespace IPs behind host physical IP. |
| **Port Forwarding (Ingress)**| `iptables -t nat -A PREROUTING -p tcp --dport <P1> -j DNAT --to-destination <IP>:<P2>` | Forwards host ports into namespace services. |