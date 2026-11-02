# DNS Resolution in Kubernetes: Architecture, FQDN Hierarchy, and Pod/Service Records

In Kubernetes, Pod and Service IP addresses are dynamically assigned. To allow applications to locate and communicate with each other without hardcoding IP addresses, Kubernetes includes an internal, cluster-wide DNS service (powered by **CoreDNS**).

This chapter explains how Kubernetes names, organizes, and resolves DNS queries for **Services** and **Pods**, the structure of **Fully Qualified Domain Names (FQDN)**, and the resolution mechanics inside a Pod's Linux network namespace.

---

## 1. High-Level Architecture: Internal DNS vs. Node DNS

It is critical to distinguish between external infrastructure DNS and internal Kubernetes DNS:

- **Node-Level DNS:** Resolves external hostnames (e.g., `api.github.com`, corporate servers). Configured in `/etc/resolv.conf` on the worker/master node itself.
- **Cluster-Internal DNS (CoreDNS):** Runs as a set of Pods inside the cluster (under the `kube-system` namespace) exposed via a static Service ClusterIP (usually `10.96.0.10`). Resolves cluster-internal objects such as `my-service.prod.svc.cluster.local`.

```text
┌────────────────────────────────────────────────────────┐
│                        POD                             │
│  Resolves: "web-service"                               │
│  Checks /etc/resolv.conf ──► nameserver 10.96.0.10     │
└───────────────────────────┬────────────────────────────┘
                            │ DNS Query (UDP/TCP Port 53)
                            ▼
┌────────────────────────────────────────────────────────┐
│              Kube-DNS Service (ClusterIP)              │
│                      10.96.0.10                        │
└───────────────────────────┬────────────────────────────┘
                            │ iptables / IPVS Forwarding
                            ▼
┌────────────────────────────────────────────────────────┐
│                   CoreDNS Pods                         │
│             (kube-system namespace)                    │
│                                                        │
│  1. Watches Kubernetes API Server for Svc / Pod events │
│  2. Resolves cluster queries from in-memory cache      │
│  3. Forwards external queries upstream (e.g., 8.8.8.8) │
└────────────────────────────────────────────────────────┘
```

---

## 2. Service DNS Records & FQDN Hierarchy

Every Kubernetes Service created in the cluster automatically gets a DNS `A` (or `AAAA` for IPv6) record pointing to its virtual `ClusterIP`.

### The Anatomy of a Service FQDN:

A Fully Qualified Domain Name (FQDN) for a Service follows a strict 5-part hierarchical naming schema:

$$\underbrace{\text{web-service}}_{\text{Service Name}} \cdot \underbrace{\text{apps}}_{\text{Namespace}} \cdot \underbrace{\text{svc}}_{\text{Object Type}} \cdot \underbrace{\text{cluster.local}}_{\text{Cluster Domain}}$$

```text
web-service . apps . svc . cluster.local
    │          │     │          │
    │          │     │          └── 1. Root Cluster Domain (default: cluster.local)
    │          │     └───────────── 2. Subdomain for Services ("svc")
    │          └─────────────────── 3. Target Namespace
    └────────────────────────────── 4. Service Object Name
```

### Resolution Rules Across Namespaces:

Assume a Service named `db-service` runs in namespace `backend`, and an application Pod runs in namespace `frontend`:

| Caller Location                  | Queried Name                           | Will it resolve? | Explanation                                                         |
| :------------------------------- | :------------------------------------- | :--------------- | :------------------------------------------------------------------ |
| Same namespace (`backend`)       | `db-service`                           | **Yes**          | Pod's search domain auto-appends `.backend.svc.cluster.local`.      |
| Different namespace (`frontend`) | `db-service`                           | **No**           | Search domain looks inside `.frontend.svc.cluster.local` and fails. |
| Different namespace (`frontend`) | `db-service.backend`                   | **Yes**          | Search domain appends `.svc.cluster.local`.                         |
| Anywhere in cluster              | `db-service.backend.svc`               | **Yes**          | Partial domain, matches cluster domain search.                      |
| Anywhere in cluster              | `db-service.backend.svc.cluster.local` | **Yes**          | **FQDN**. Resolves directly without searching.                      |

---

## 3. Pod DNS Records: Mechanics & Formatting

Unlike Services, Pods do **not** get a simple DNS record using their Pod name by default.

### Why?

Pods in a Deployment or ReplicaSet are disposable and possess random hashes in their names (e.g., `web-deploy-7bf488c89-x8j2l`). Creating dynamic records for every randomly generated Pod name would bloat the DNS server's cache.

### Standard Pod DNS Record Format:

When Pod DNS resolution is enabled, CoreDNS generates a record based on the **Pod's IP address**, with the dots (`.`) replaced by dashes (`-`):

$$\underbrace{\text{10-244-1-5}}_{\text{Dashed Pod IP}} \cdot \underbrace{\text{default}}_{\text{Namespace}} \cdot \underbrace{\text{pod}}_{\text{Object Type}} \cdot \underbrace{\text{cluster.local}}_{\text{Cluster Domain}}$$

- If a Pod has IP `10.244.2.14` in namespace `prod`, its DNS name is:  
  `10-244-2-14.prod.pod.cluster.local`

---

## 4. Headless Services & Individual Pod DNS Names

When you run stateful distributed workloads (such as Elasticsearch, Kafka, Cassandra, or MongoDB via `StatefulSet`), each Pod must have a **predictable, persistent hostname** (e.g., `db-0`, `db-1`), rather than a dashed IP.

This is achieved using a **Headless Service** (`clusterIP: None`) combined with Pod `hostname` and `subdomain` fields.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None # Makes it a Headless Service
  selector:
    app: database
  ports:
    - port: 5432
---
apiVersion: v1
kind: Pod
metadata:
  name: db-node-0
  labels:
    app: database
spec:
  hostname: db-0 # Hostname inside the pod
  subdomain: db-headless # Must match the Headless Service name
  containers:
    - name: postgres
      image: postgres
```

### Resulting DNS Record:

CoreDNS creates a direct `A` record pointing to the Pod's real IP:

$$\text{db-0} \cdot \text{db-headless} \cdot \text{default} \cdot \text{svc} \cdot \text{cluster.local}$$

This guarantees that `db-0` always resolves to that specific replica's current IP address, even if the Pod dies and gets rescheduled onto another node.

---

## 5. How Pods Query DNS: The `/etc/resolv.conf` File

When `kubelet` provisions a Pod's network namespace, it automatically injects a tailored `/etc/resolv.conf` file:

```ini
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### Directive Breakdown:

1. **`nameserver 10.96.0.10`:**  
   Directs all DNS queries to the `kube-dns` Service IP (CoreDNS).
2. **`search ...` (Search Domains):**  
   If a client requests an unqualified name (e.g. `curl db-service`), the OS resolver appends each search domain in order until it gets a hit:
   1. `db-service.default.svc.cluster.local` (Hit! Query succeeds)
   2. `db-service.svc.cluster.local`
   3. `db-service.cluster.local`
3. **`options ndots:5`:**  
   If a domain contains fewer than 5 dots, the Linux resolver treats it as an incomplete relative domain and queries the `search` paths **before** attempting an external lookup.

---

## 6. Verification and Diagnostic Commands (CKA Toolkit)

### 1. View Cluster DNS Service IP

```bash
kubectl get svc -n kube-system -l k8s-app=kube-dns
```

_Note the `CLUSTER-IP` (usually `10.96.0.10`)._

### 2. Inspect a Running Pod's DNS Configuration

```bash
kubectl exec <pod-name> -- cat /etc/resolv.conf
```

### 3. Test DNS Resolution using a Debug Pod (`nslookup` / `dig`)

```bash
# Run a temporary pod with DNS network tools:
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup web-service
```

- **Query a cross-namespace service:**
  ```bash
  nslookup web-service.apps.svc.cluster.local
  ```
- **Query a Pod directly:**
  ```bash
  nslookup 10-244-1-5.default.pod.cluster.local
  ```

---

## 7. Summary Comparison: Service vs. Pod DNS

| Feature           | Service DNS Record              | Pod DNS Record (Default)               | Pod DNS Record (Headless)                    |
| :---------------- | :------------------------------ | :------------------------------------- | :------------------------------------------- |
| **Object Domain** | `.svc.cluster.local`            | `.pod.cluster.local`                   | `.svc.cluster.local`                         |
| **Prefix Format** | `<service-name>`                | `<ip-with-dashes>`                     | `<pod-hostname>.<subdomain>`                 |
| **Example FQDN**  | `web.default.svc.cluster.local` | `10-244-1-5.default.pod.cluster.local` | `db-0.db-headless.default.svc.cluster.local` |
| **Resolves to**   | Virtual `ClusterIP`             | Real Pod IP                            | Real Pod IP                                  |
| **Automatic?**    | Yes, on Service creation        | Yes (if enabled in CNI/DNS)            | Requires `clusterIP: None` + `subdomain`     |
