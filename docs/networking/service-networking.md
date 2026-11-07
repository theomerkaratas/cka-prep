# Kubernetes Service Networking: Architecture, kube-proxy, and iptables Mechanics

In Kubernetes, Pods are ephemeral; they are created, scaled, and destroyed dynamically, and their IP addresses change frequently. If client applications relied directly on Pod IP addresses, every Pod restart or deployment rollout would break network connectivity.

A **Service** provides a stable, persistent virtual IP (ClusterIP) and DNS name that acts as a front-end abstraction over a dynamic group of backend Pods.

Unlike a Pod, a Service is **not a container, not a Linux network namespace, and not a network interface**. There is no process listening on a Service's IP. A Service is purely an in-memory virtual object inside the Kubernetes API that gets translated into **kernel-level packet forwarding rules** on every node in the cluster.

---

## 1. High-Level Architecture: How Services Actually Work

To understand how traffic reaches a Service, observe the interaction between the Control Plane, `kube-proxy`, and the Linux kernel:

```text
┌────────────────────────────────────────────────────────┐
│                   kube-apiserver                       │
│  Stores Service & EndpointSlice definitions in etcd    │
└───────────────────────────┬────────────────────────────┘
                            │ Watch API (Service / Endpoints changes)
                            ▼
┌────────────────────────────────────────────────────────┐
│               kube-proxy (on every node)               │
│  Daemon that synchronizes Service state to the kernel  │
└───────────────────────────┬────────────────────────────┘
                            │ Writes packet translation rules
                            ▼
┌────────────────────────────────────────────────────────┐
│                   Linux Kernel                         │
│           (iptables NAT table / IPVS)                  │
│                                                        │
│  Match:    DST = 10.96.0.10:80 (Virtual ClusterIP)     │
│  Action:   DNAT -> 10.244.1.5:8080 (Real Pod IP)       │
└────────────────────────────────────────────────────────┘
```

### Key Differences: Pod Networking vs. Service Networking

| Attribute | Pod Networking | Service Networking |
| :--- | :--- | :--- |
| **Backing Entity** | Real Linux Network Namespace & `veth` pair | Virtual abstraction (No interface, no namespace) |
| **Process Listening?** | Yes (e.g., NGINX, Python, Go daemon) | No process listens on the Service IP |
| **Routing / Forwarding** | L3 routing / CNI overlay encapsulation | L4 Destination NAT (DNAT) via `iptables` / IPVS |
| **Managed By** | CNI Plugin (`bridge`, `calico`, `flannel`) | `kube-proxy` + Linux Kernel |
| **Scope** | Node-local bridge + inter-node routing | Cluster-wide forwarding rules programmed on **all** nodes |

---

## 2. IP Range Separation: Pod CIDR vs. Service ClusterIP CIDR

Kubernetes enforces a strict separation between two independent IP address pools:

1. **Pod Network CIDR (`--cluster-cidr`):**
   * Configured in `kube-controller-manager`.
   * Real, routable IP addresses assigned directly to Pod network namespaces (e.g., `10.244.0.0/16`).
2. **Service ClusterIP CIDR (`--service-cluster-ip-range`):**
   * Configured in `kube-apiserver`.
   * Completely virtual IP addresses allocated to Services (e.g., `10.96.0.0/12`).

```text
┌───────────────────────────────────────────────────────────────┐
│                     Cluster IP Address Space                  │
├───────────────────────────────┬───────────────────────────────┤
│    Pod Network CIDR           │   Service ClusterIP CIDR      │
│    (e.g., 10.244.0.0/16)      │   (e.g., 10.96.0.0/12)        │
│                               │                               │
│  Node 01: 10.244.1.0/24       │  Kubernetes DNS: 10.96.0.10   │
│  Node 02: 10.244.2.0/24       │  App Service:    10.103.1.50  │
└───────────────────────────────┴───────────────────────────────┘
```

> [!IMPORTANT]
> The **Pod CIDR** and the **Service ClusterIP CIDR** must **never overlap**.  
> If they overlap, packet routing rules will conflict, causing traffic meant for a Service to be misrouted directly to a Pod, or vice versa.

---

## 3. The Role of `kube-proxy`

Every node in a Kubernetes cluster runs `kube-proxy` (typically deployed as a `DaemonSet` in the `kube-system` namespace).

### What `kube-proxy` Does:
1. Opens a persistent connection (`WATCH`) to `kube-apiserver`.
2. Whenever a `Service` or `Endpoints` / `EndpointSlice` object is created, updated, or deleted, `kube-proxy` receives an event.
3. It takes the Service IP (`ClusterIP`), target port, and backend Pod IPs, and programs packet manipulation rules directly into the host's Linux kernel.

### Supported Proxy Modes:

`kube-proxy` can operate in three distinct modes, configured via `--proxy-mode`:

```text
1. userspace (Legacy - Obsolete):
   Packets switch between kernel space and user space kube-proxy process. Very slow.

2. iptables (Default):
   kube-proxy programs Netfilter/iptables rules. Packet routing and load balancing
   occur entirely in kernel space via random probability chains.

3. ipvs (High Performance):
   kube-proxy programs Linux IPVS (IP Virtual Server). Uses hash tables instead of
   sequential lists; scales efficiently to tens of thousands of services (O(1) lookup).
```

---

## 4. How `iptables` Mode Implements Service Forwarding (Under the Hood)

When a client Pod sends a packet to a Service ClusterIP (`10.103.132.104:3306`), the packet enters the host kernel network stack.

Because no interface owns `10.103.132.104`, why is the packet not dropped?
Because `kube-proxy` inserted **Netfilter `PREROUTING` and `OUTPUT` rules** that intercept the packet before routing decisions are made.

### The `iptables` Rule Chain Walkthrough

`kube-proxy` organizes its rules into hierarchical custom chains inside the **`nat`** table:

```text
Packet to 10.103.132.104:3306
           │
           ▼
     [PREROUTING / OUTPUT]
           │
           ▼
     [KUBE-SERVICES] ────────── Match Service IP & Port
           │
           ▼
     [KUBE-SVC-XXXXX] ───────── Load Balancing (Random Probability)
      ├── 50% ──► [KUBE-SEP-AAAA] ──► DNAT to Pod 1 (10.244.1.2:3306)
      └── 50% ──► [KUBE-SEP-BBBB] ──► DNAT to Pod 2 (10.244.2.4:3306)
```

1. **`KUBE-SERVICES` Chain:**
   Checks if the destination IP matches any known Service ClusterIP.
   ```text
   -A KUBE-SERVICES -d 10.103.132.104/32 -p tcp --dport 3306 -j KUBE-SVC-DB
   ```
2. **`KUBE-SVC-*` Chain (Service Chain):**
   Handles load balancing across backend endpoints. If there are 2 Pods, it uses the `statistic` module:
   ```text
   # 1st rule: 50% probability to jump to 1st endpoint
   -A KUBE-SVC-DB -m statistic --mode random --probability 0.5000000000 -j KUBE-SEP-01
   # 2nd rule: Remaining 50% falls through to 2nd endpoint
   -A KUBE-SVC-DB -j KUBE-SEP-02
   ```
3. **`KUBE-SEP-*` Chain (Service Endpoint Chain):**
   Applies **Destination NAT (DNAT)**, rewriting the packet's destination IP from the Service Virtual IP to the real Pod IP:
   ```text
   -A KUBE-SEP-01 -p tcp -m tcp -j DNAT --to-destination 10.244.1.2:3306
   ```

Once DNAT occurs, standard Linux routing forwards the packet across the CNI network to the destination Pod.

---

---

## 5. Kubernetes Service Types & Creation Methods

Kubernetes supports 4 primary Service types, each designed for a specific networking scope:

| Service Type | Scope / Access Level | Port Range / Allocation | Typical Use Case |
| :--- | :--- | :--- | :--- |
| **`ClusterIP`** (Default) | Internal to cluster only | Cluster-internal virtual IP from `--service-cluster-ip-range` | Inter-pod communication (e.g., backend API, database) |
| **`NodePort`** | External access via Node IP | Port **`30000 - 32767`** opened on all cluster nodes | Direct testing, bare-metal clusters without cloud LBs |
| **`LoadBalancer`** | External access via Cloud LB | Provisions a cloud provider LB (AWS NLB, GCP LB, Azure LB) | Production web traffic facing the public internet |
| **`ExternalName`** | Internal DNS redirection | No ClusterIP, no Endpoints, returns a `CNAME` record | Accessing external services (e.g., `db.aws.com`) via a local K8s DNS name |

---

### A. ClusterIP (Default)

Provides a stable internal IP accessible only from within the cluster.

#### 1. Imperative Command (Fastest for CKA):
```bash
# Expose an existing pod / deployment:
kubectl expose pod web-pod --name=web-service --port=80 --target-port=8080 --type=ClusterIP

# Or create standalone without an existing pod:
kubectl create service clusterip web-service --tcp=80:8080
```
*(Format: `--tcp=<service-port>:<target-pod-port>`)*

#### 2. Declarative YAML Manifest:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP          # Optional (default is ClusterIP)
  selector:
    app: web-app           # Matches labels on backend Pods
  ports:
    - protocol: TCP
      port: 80             # Port exposed by the Service (ClusterIP)
      targetPort: 8080     # Port listening inside the Pod container
```

---

### B. NodePort

Exposes the Service on each Node's external/internal IP at a static port from the **`30000-32767`** range.

#### 1. Imperative Command (Fastest for CKA):
```bash
# Expose an existing pod / deployment:
kubectl expose deployment my-deploy --name=my-nodeport-svc --type=NodePort --port=80 --target-port=8080

# Or create standalone (Kubernetes automatically picks a free nodePort in 30000-32767):
kubectl create service nodeport my-nodeport-svc --tcp=80:8080
```

#### 2. Declarative YAML Manifest:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-svc
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
    - protocol: TCP
      port: 80             # Service ClusterIP port
      targetPort: 8080     # Port inside the Pod
      nodePort: 30080      # (Optional) If omitted, K8s auto-assigns from 30000-32767
```

---

### C. LoadBalancer

Builds on top of `NodePort` and automatically provisions an external cloud load balancer (AWS, GCP, Azure, OpenStack) that routes traffic to the NodePorts.

#### 1. Imperative Command:
```bash
kubectl expose deployment frontend --name=frontend-lb --type=LoadBalancer --port=443 --target-port=8443
# Or:
kubectl create service loadbalancer frontend-lb --tcp=80:8080
```

#### 2. Declarative YAML Manifest:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---

### D. ExternalName

A special Service type that does not have selectors, does not allocate a `ClusterIP`, and does not create endpoints. Instead, CoreDNS returns a **CNAME** record pointing to an external DNS name.

#### 1. Imperative Command:
```bash
kubectl create service externalname my-ext-db --external-name=database.external-company.com
```

#### 2. Declarative YAML Manifest:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-ext-db
spec:
  type: ExternalName
  externalName: database.external-company.com
```
*When a Pod queries `my-ext-db.default.svc.cluster.local`, CoreDNS immediately answers with the external CNAME `database.external-company.com`.*

---

### E. Headless Service (ClusterIP: None)

When you need direct, individual Pod IP resolution (common in StatefulSets like Kafka, Cassandra, MongoDB) without load balancing or proxying through a single ClusterIP.

#### YAML Manifest:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None          # This defines it as a Headless Service
  selector:
    app: db
  ports:
    - port: 5432
      targetPort: 5432
```
*DNS behavior: Querying `db-headless` returns the individual A-records of all matching Pods directly, bypassing kube-proxy.*

---

## 6. Verification and Inspection Commands

### 1. View Service and Associated Endpoints
```bash
# List services and their ClusterIPs
kubectl get svc

# Inspect the backend Pod IPs attached to the Service
kubectl get endpoints <service-name>
# or modern EndpointSlices:
kubectl get endpointslices
```

### 2. Inspect `kube-apiserver` Service IP Range
```bash
# On the Control Plane node:
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
```

### 3. Inspect `kube-proxy` Mode and Logs
```bash
# Check the proxyMode in kube-proxy ConfigMap:
kubectl get cm kube-proxy -n kube-system -o yaml | grep mode

# View kube-proxy logs on any node:
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50
```

### 4. Inspect iptables Rules Directly on a Node
```bash
# List all Service NAT rules:
sudo iptables -t nat -L KUBE-SERVICES -n -v

# Search for rules belonging to a specific service:
sudo iptables -t nat -S | grep <service-name>
```

---

## 7. Summary Cheat Sheet

| Concept | Explanation |
| :--- | :--- |
| **What is a Service?** | An in-memory API abstraction mapped to kernel packet-filtering rules. |
| **Does Service have an interface?** | **No.** It has no MAC address, no network namespace, and no listening daemon. |
| **Who programs the rules?** | `kube-proxy` daemon running on each node. |
| **How does it forward traffic?** | Performs Destination NAT (DNAT) via `iptables` or IPVS. |
| **Service CIDR Flag** | `--service-cluster-ip-range` on `kube-apiserver`. |
| **Pod CIDR Flag** | `--cluster-cidr` on `kube-controller-manager`. |

