# Kubernetes Cluster Networking: Node Prerequisites, Port Requirements, and Diagnostics

A Kubernetes cluster is a distributed system consisting of **Control Plane (Master) Nodes** and **Worker Nodes**. Before deploying Pods or installing a CNI plugin, the underlying physical or virtual machines must satisfy strict operating system and network topology prerequisites.

Below is a technical breakdown of cluster networking requirements, mandatory listening ports, communication flows, and diagnostic commands tested in the **CKA (Certified Kubernetes Administrator)** exam.

---

## 1. Node-Level Network Prerequisites

Before joining any machine into a cluster (via `kubeadm init` or `kubeadm join`), each node must meet three fundamental networking requirements:

### A. Network Interface & IP Connectivity
* Every node must have at least one network interface connected to the network.
* The interface must have a valid IPv4 or IPv6 address statically assigned or leased via a persistent DHCP reservation.
* **Full mesh connectivity:** Every node in the cluster must be able to route IP traffic to every other node without Network Address Translation (NAT) between node IPs.

### B. Unique Hostnames
* Every node must have a **unique hostname**.
* If two nodes share the same hostname, the `kubelet` registrations will overwrite each other in the Kubernetes API, causing one node to constantly flip to `NotReady`.
* Check or set the hostname:
  ```bash
  hostnamectl status
  hostnamectl set-hostname node01
  ```

### C. Unique MAC Addresses & Product UUIDs
* Virtual machine cloning often duplicates network card MAC addresses or hardware UUIDs.
* Kubernetes uses the MAC address and `/sys/class/dmi/id/product_uuid` to uniquely identify physical/virtual interfaces.
* Inspect with:
  ```bash
  ip link show
  sudo cat /sys/class/dmi/id/product_uuid
  ```
  If MAC addresses or product UUIDs collide, nodes will fail to register properly with the API server.

---

## 2. Cluster Port Requirements & Firewall Rules

Kubernetes components communicate over dedicated TCP ports. If local firewalls (`ufw`, `firewalld`, `iptables`) or cloud security groups block these ports, the control plane will degrade or workers will fail to join.

```text
                  ┌────────────────────────────────────────────────────────┐
                  │                 CONTROL PLANE (MASTER)                 │
                  │                                                        │
                  │  ┌───────────────────────────┐                         │
                  │  │ kube-apiserver: 6443      │◄───────┐                │
                  │  └─────────────┬─────────────┘        │                │
                  │                │                      │                │
                  │  ┌─────────────▼─────────────┐        │                │
                  │  │ etcd: 2379 (client)       │        │                │
                  │  │ etcd: 2380 (peer)         │        │                │
                  │  └───────────────────────────┘        │                │
                  │                                       │                │
                  │  ┌───────────────────────────┐        │                │
                  │  │ kube-scheduler: 10259     │        │                │
                  │  │ kube-controller-mgr: 10257│        │                │
                  │  │ kubelet API: 10250        │        │                │
                  │  └───────────────────────────┘        │                │
                  └───────────────────────────────────────┼────────────────┘
                                                          │
                                                          │ 6443 (Join & Sync)
                                                          │ 10250 (Logs/Exec)
                                                          │
                  ┌───────────────────────────────────────┼────────────────┐
                  │                 WORKER NODE           │                │
                  │                                       │                │
                  │  ┌───────────────────────────┐        │                │
                  │  │ kubelet API: 10250        │◄───────┘                │
                  │  └───────────────────────────┘                         │
                  │                                                        │
                  │  ┌───────────────────────────┐                         │
                  │  │ NodePort Services:        │◄── (External Clients)   │
                  │  │ 30000 - 32767             │                         │
                  │  └───────────────────────────┘                         │
                  └────────────────────────────────────────────────────────┘
```

---

### A. Control Plane (Master) Nodes

| Protocol | Port Range | Component | Direction / Purpose |
| :--- | :--- | :--- | :--- |
| **TCP** | **`6443`** | `kube-apiserver` | **Inbound from everywhere.** The front door of the cluster. Worker nodes (`kubelet`, `kube-proxy`), operators, and `kubectl` clients connect here. |
| **TCP** | **`2379`** | `etcd` Client API | **Inbound from control plane.** Used by `kube-apiserver` to read and write cluster state. Must never be exposed to worker nodes or public networks. |
| **TCP** | **`2380`** | `etcd` Peer Communication | **Inbound between master nodes.** Used only when running High Availability (HA) multi-master clusters for etcd Raft consensus synchronization. |
| **TCP** | **`10250`** | `kubelet` API | **Inbound from `kube-apiserver`.** The API server connects to the master's local `kubelet` to pull pod logs, attach to containers, and run `kubectl exec`. |
| **TCP** | **`10259`** | `kube-scheduler` | **Local control loop.** Secure metrics and health endpoints. |
| **TCP** | **`10257`** | `kube-controller-manager` | **Local control loop.** Secure metrics and health endpoints. |

> **Note on legacy ports:** In older Kubernetes versions, `kube-scheduler` listened on unauthenticated HTTP port `10251` and `kube-controller-manager` on `10252`. Current production versions run securely over HTTPS on **`10259`** and **`10257`**.

---

### B. Worker Nodes

| Protocol | Port Range | Component | Direction / Purpose |
| :--- | :--- | :--- | :--- |
| **TCP** | **`10250`** | `kubelet` API | **Inbound from Control Plane (`kube-apiserver`).** Essential for `kubectl logs`, `kubectl exec`, and `kubectl attach` to work against Pods running on this worker. |
| **TCP/UDP**| **`30000 - 32767`** | NodePort Services | **Inbound from external clients / load balancers.** When a Kubernetes Service of type `NodePort` is created, `kube-proxy` opens a listening port on every worker node in this default range. |

---

## 3. Communication Paths Explained

Understanding which component initiates the connection clarifies why firewall rules matter:

### 1. Worker to Master (`kubelet` -> `kube-apiserver:6443`)
* **Direction:** Outbound from Worker, Inbound on Master `6443`.
* Every `kubelet` and `kube-proxy` continuously polls the API server for spec updates (new Pods to run, iptables rules to update).
* If port `6443` is unreachable from a worker, that worker transitions to `NotReady`.

### 2. Master to Worker (`kube-apiserver` -> `kubelet:10250`)
* **Direction:** Outbound from Master, Inbound on Worker `10250`.
* When an administrator runs `kubectl logs <pod-name>`, the client talks to `kube-apiserver:6443`. The API server then initiates an outbound connection directly to the target worker node on port **`10250`**.
* If port `10250` is blocked on the worker, the cluster operates normally, but `kubectl logs` and `kubectl exec` will fail with connection timeout errors.

### 3. Service Exposure (`Client` -> `NodePort:30000-32767`)
* When routing external traffic into the cluster via `NodePort`, traffic hits any worker node's physical IP on the allocated port in this range.
* `kube-proxy` forwards this ingress packet to the target Pod IP (which may reside on the same node or a completely different node across the CNI network).

---

## 4. Network Diagnostic & Verification Commands

When troubleshooting node readiness or cluster join failures, use these tools to isolate the layer causing the issue:

### Inspecting Hardware & IP Layer (Layer 2 & 3)
```bash
# Check interface states (UP/DOWN) and MAC addresses
ip link

# View assigned IP addresses
ip addr

# Add an IP address dynamically to an interface
ip addr add 192.168.1.10/24 dev eth0

# View neighbor discovery / ARP cache (verify Layer 2 resolution between nodes)
ip neigh
# or legacy:
arp -n
```

### Inspecting Routing & Transit
```bash
# Display the kernel routing table
ip route
# or legacy:
route -n

# Add a static route to reach another node's subnet via a gateway
ip route add 192.168.1.0/24 via 192.168.2.1

# Verify that kernel packet forwarding is active (Mandatory for K8s nodes)
cat /proc/sys/net/ipv4/ip_forward
# Expected output: 1
```

### Inspecting Listening Ports & Sockets
To verify whether Kubernetes control plane components or the `kubelet` are actually running and listening on their designated ports:

```bash
# List all TCP listening ports with process names and numeric IPs
netstat -plnt
# or using the modern, faster ss utility:
ss -tulpn
```

* Filtering for specific Kubernetes ports:
  ```bash
  # Check if kube-apiserver is listening on 6443:
  ss -tulpn | grep 6443

  # Check if kubelet is listening on 10250:
  ss -tulpn | grep 10250

  # Check etcd ports:
  ss -tulpn | grep -E '2379|2380'
  ```

### Testing Remote Port Reachability
From a worker node, test if the control plane ports are open and unblocked by firewalls:

```bash
# Using netcat:
nc -z -v <master-ip> 6443

# Using curl:
curl -k https://<master-ip>:6443/healthz
# Expected response: ok
```

---

## 5. Summary Cheat Sheet: Ports & Rules

| Role | Port | Process | Access Scope |
| :--- | :--- | :--- | :--- |
| **Master** | `6443` | `kube-apiserver` | All nodes, administrators, CI/CD |
| **Master** | `2379` | `etcd` (client) | `kube-apiserver` only |
| **Master** | `2380` | `etcd` (peers) | Other control plane nodes only |
| **Master** | `10259` | `kube-scheduler` | Localhost / Control plane internal |
| **Master** | `10257` | `kube-controller-manager` | Localhost / Control plane internal |
| **Master & Worker** | `10250` | `kubelet` API | `kube-apiserver` only |
| **Worker** | `30000-32767` | NodePort Services | External users, Load Balancers |