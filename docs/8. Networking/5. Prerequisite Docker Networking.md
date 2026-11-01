# Docker Networking: Architecture, Linux Namespaces, and Packet Routing

Docker networking is an automated abstraction built directly on the Linux kernel primitives discussed in previous sections: **Network Namespaces**, **Virtual Ethernet (`veth`) pairs**, **Linux Bridges**, and **`iptables` packet filtering/NAT**.

When the Docker daemon (`dockerd`) provisions containers, it manipulates these underlying OS features to implement isolation, local switching, and external routing.

---

## 1. Docker Network Drivers Overview

When launching a container, Docker provides three primary built-in network drivers:

### A. The `none` Driver (`docker run --network none`)
* **Mechanism:** Docker creates a new, isolated network namespace for the container, but **does not configure any network interfaces** inside it other than the loopback (`lo`) interface.
* **Result:** The container has no IP address, no default route, and no connection to any virtual switch or the host. It cannot send or receive traffic across any network.
* **Use Case:** Batch computations, air-gapped security workloads, or self-contained processing tasks that do not require external I/O.

### B. The `host` Driver (`docker run --network host`)
* **Mechanism:** Docker **does not create a separate network namespace**. The container process executes inside the host's root network namespace.
* **Result:**
  * The container shares the host's network interfaces (`eth0`, `lo`), IP addresses, routing table, and ARP cache.
  * If an application inside the container binds to port `80`, it immediately binds to port `80` on the host's physical network interface without requiring port mapping flags (`-p`).
* **Limitation (Port Collisions):** Two containers configured with `--network host` cannot bind to the same port simultaneously; the second process fails with `bind: address already in use`.

### C. The `bridge` Driver (`docker run --network bridge`)
* **Mechanism:** The **default mode**. Docker creates an independent network namespace for each container and connects them to an internal virtual switch on the host called `docker0`.
* **Result:** Containers receive private IP addresses from an internal subnet (`172.17.0.0/16`), maintain isolated network stacks, and communicate with external networks through Network Address Translation (NAT).

---

## 2. The Default Bridge Architecture (`docker0`)

When the Docker daemon starts on a Linux host, it inspects the system to see if a default bridge exists. If not, it creates one.

```text
Host System (Root Network Namespace)
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Physical Interface: eth0 (192.168.1.10)                   │
│                                                             │
│   Virtual Bridge:     docker0 (172.17.0.1/16)               │
│                       [Layer 2 Virtual Switch]              │
│                       ▲                  ▲                  │
└───────────────────────┼──────────────────┼──────────────────┘
                        │ veth pair        │ veth pair
                        ▼                  ▼
┌───────────────────────────────┐  ┌───────────────────────────────┐
│ Container 1 Namespace         │  │ Container 2 Namespace         │
│ eth0: 172.17.0.2/16           │  │ eth0: 172.17.0.3/16           │
│ Default GW: 172.17.0.1        │  │ Default GW: 172.17.0.1        │
└───────────────────────────────┘  └───────────────────────────────┘
```

### Nomenclature: Docker vs. Linux Kernel
* In Docker CLI output (`docker network ls`), this network is listed with the name **`bridge`**.
* In Linux kernel output (`ip link` / `ip addr`), the network interface is named **`docker0`**.
* They represent the exact same entity: `docker0` is the underlying Linux software bridge created via:
  ```bash
  ip link add docker0 type bridge
  ```

### The Dual Role of `docker0`
1. **Layer 2 Switch for Containers:** All containers on the bridge have their virtual network interfaces plugged directly into `docker0`. Traffic between containers on the same host flows through `docker0` via MAC address lookups.
2. **Layer 3 Gateway for Containers:** The bridge is assigned an IP address on the host (typically `172.17.0.1/16`). From the perspective of the containers, `172.17.0.1` is their **Default Gateway**.

---

## 3. Container Creation and Network Plumbing Step-by-Step

When `docker run -d --name web nginx` is executed, the Docker daemon performs the following automated steps under the hood:

### Step 1: Network Namespace Allocation
Docker creates a new network namespace for the container. 
* By default, Docker does not store namespace handles in `/var/run/netns/` (the path standard `ip netns` reads), but in `/var/run/docker/netns/`.
* To inspect Docker namespaces with standard `ip netns` commands, create a symbolic link:
  ```bash
  mkdir -p /var/run/netns
  ln -s /var/run/docker/netns/<netns_id> /var/run/netns/<container_name>
  ```
* Alternatively, extract the container's Process ID (`PID`) and query its namespace directly:
  ```bash
  PID=$(docker inspect --format '{{.State.Pid}}' web)
  nsenter -t $PID -n ip addr
  ```

### Step 2: Virtual Wire (`veth` Pair) Generation
Docker creates a pair of peer virtual Ethernet interfaces:
```bash
ip link add veth<id> type veth peer name eth0
```

### Step 3: Interface Placement and Naming
* One end of the pair (`veth<id>`) remains in the host's root namespace and is attached to the bridge:
  ```bash
  ip link set veth<id> master docker0
  ip link set dev veth<id> up
  ```
* The peer end is moved into the container's network namespace and renamed to **`eth0`**:
  ```bash
  ip link set dev eth0 netns <container_netns>
  ```

### Step 4: IP Allocation and Default Route
Docker’s embedded IP Address Management (IPAM) driver assigns the next available IP from the `172.17.0.0/16` subnet to the container's `eth0` and sets the default route:
```bash
# Executed inside the container's namespace:
ip addr add 172.17.0.2/16 dev eth0
ip link set dev eth0 up
ip route add default via 172.17.0.1
```

### Interface Index Pairing
On the host, running `ip link` shows interfaces with names like `veth3a9e2bf@if8`. The `@if8` notation indicates that this interface is paired with interface index `8` located inside the container's namespace.

---

## 4. Traffic Flow Patterns on the Docker Bridge

### Pattern 1: Container-to-Container (Same Host)
* **Scenario:** Container 1 (`172.17.0.2`) sends a packet to Container 2 (`172.17.0.3`).
* **Flow:**
  1. Container 1 checks its subnet mask (`/16`) and confirms `172.17.0.3` is on the same local network.
  2. Container 1 sends an ARP request for `172.17.0.3` out of its `eth0`.
  3. The frame enters the `veth` pair and reaches `docker0`.
  4. `docker0` operates as a Layer 2 switch: it forwards the ARP frame to the port connected to Container 2.
  5. Container 2 replies with its MAC address.
  6. Data frames are switched directly across `docker0` without invoking host-level routing.

---

### Pattern 2: Container-to-External Network (Egress)
* **Scenario:** Container 1 (`172.17.0.2`) connects to an internet server (`142.250.180.206`).
* **Flow:**
  1. Container 1 checks its routing table; the target IP is outside `172.17.0.0/16`.
  2. The packet is sent to the default gateway: `172.17.0.1` (`docker0`).
  3. The packet leaves the container's namespace and enters the host's root namespace via `docker0`.
  4. The host kernel evaluates its own routing table: the route to the internet points out of physical interface `eth0`.
  5. **Kernel IP Forwarding:** Since `net.ipv4.ip_forward = 1`, the kernel forwards the packet from `docker0` to `eth0`.
  6. **Source NAT (Masquerading):** Outbound packets have a private source IP (`172.17.0.2`). External routers cannot route back to this private address. Docker adds an `iptables` rule to masquerade the traffic:
     ```bash
     iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
     ```
     The kernel rewrites the Source IP from `172.17.0.2` to the host's physical IP (`192.168.1.10`).
  7. When the remote server replies to `192.168.1.10`, the host's connection tracking table (`conntrack`) de-NATs the packet and returns it to `172.17.0.2`.

---

## 5. Exposing Services: Port Mapping (`-p 8080:80`)

By default, the `172.17.0.0/16` subnet is private and accessible only from the host itself and other containers on the same bridge. External systems on the physical LAN (`192.168.1.0/24`) cannot reach `172.17.0.2` directly because upstream physical routers have no route pointing that subnet to the Docker host.

To make a container service available externally, Docker uses **Port Publishing**:

```bash
docker run -d -p 8080:80 --name web nginx
```

### How Docker Implements Port Mapping
Docker accomplishes port mapping using **Destination NAT (DNAT)** in Linux `iptables`.

When Docker starts, it creates custom user-defined chains in the `iptables` NAT table:
* `PREROUTING` -> jumps to `DOCKER` chain.
* `OUTPUT` -> jumps to `DOCKER` chain (for locally generated traffic on the host).

When `-p 8080:80` is specified, Docker injects a DNAT rule into the `DOCKER` chain:

```bash
iptables -t nat -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```

### Inspecting the Rules
To view the exact rules created by Docker:

```bash
iptables -t nat -L DOCKER -n -v
```

Output:
```text
Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source      destination         
    0     0 DNAT       tcp  --  !docker0 *     0.0.0.0/0   0.0.0.0/0   tcp dpt:8080 to:172.17.0.2:80
```

### End-to-End Ingress Packet Walkthrough
1. An external client (`192.168.1.50`) sends an HTTP request to `http://192.168.1.10:8080`.
2. The packet arrives on the host's `eth0` interface:
   * **Source:** `192.168.1.50:54321`
   * **Destination:** `192.168.1.10:8080`
3. Before the kernel makes a routing decision, the packet hits the `PREROUTING` chain in the `nat` table, which routes it into the `DOCKER` chain.
4. The `DOCKER` chain rule matches: `Destination Port == 8080` arriving on any interface except `docker0`.
5. **DNAT Execution:** The kernel rewrites the Destination IP and Port:
   * **New Destination:** `172.17.0.2:80`
6. The kernel consults its routing table: `172.17.0.2` belongs to the `docker0` network link.
7. The kernel forwards the packet across `docker0` through the `veth` pair into the container's namespace.
8. Nginx receives the packet on local port `80`, processes the HTTP request, and returns the response.

---

## 6. Docker Networking vs. Underlying Linux Mechanisms

| Docker Concept / Command               | Underlying Linux Kernel Primitive              | Purpose                                                           |
| :------------------------------------- | :--------------------------------------------- | :---------------------------------------------------------------- |
| **`docker network create ...`**        | `ip link add ... type bridge`                  | Allocates a new virtual Layer 2 switch on the host.               |
| **Container network isolation**        | `CLONE_NEWNET` / Network Namespace             | Isolates network interfaces, routing tables, and sockets.         |
| **Connecting container to bridge**     | `veth` pair + `ip link set dev ... master`     | Acts as a virtual patch cable between namespace and bridge.       |
| **Container IP assignment**            | `ip addr add <IP>/<CIDR> dev eth0`             | Configures internal Layer 3 addressing inside the namespace.      |
| **Outbound Internet Access**           | `iptables -t nat -A POSTROUTING -j MASQUERADE` | Rewrites private container IPs to host IP for egress.             |
| **Port Mapping (`-p host:container`)** | `iptables -t nat -A DOCKER -j DNAT`            | Rewrites inbound host destination ports to container IP:Port.     |
| **`--network host`**                   | Shares Root Network Namespace                  | Bypasses network isolation entirely for zero-overhead networking. |
| **`--network none`**                   | Unplumbed Network Namespace                    | Completely disconnects the container process from all networks.   |