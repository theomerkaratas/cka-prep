# CoreDNS Architecture and Mechanics in Kubernetes

In earlier sections, we examined how standalone CoreDNS functions using a `Corefile` and plugin pipelines. In Kubernetes (starting from v1.12), **CoreDNS** is the standard, authoritative cluster DNS server.

This document provides a comprehensive, technical walkthrough of how CoreDNS is deployed inside a cluster, how its configuration is mounted, how the `kubernetes` plugin operates, and how `kubelet` automatically connects Pods to the DNS infrastructure.

---

## 1. The Core Problem: Why Does Kubernetes Need Internal DNS?

In a standard Linux server environment, if one machine needs to reach another machine by name, you have two options:

1. **Static resolution:** Manually write hostnames and IPs into `/etc/hosts` (e.g. `10.244.1.5 web-service`).
2. **Dynamic resolution:** Set up a central DNS server and write its IP address into `/etc/resolv.conf`.

In Kubernetes, Pods and Services are ephemeral:

- Hundreds of Pods are created, rescheduled, and terminated every minute.
- Pod IP addresses change continuously.
- It is impossible to manually update `/etc/hosts` inside dynamic containers.

Therefore, Kubernetes deploys a **centralized, cluster-internal DNS solution**. Every time a Service or Pod is created, the DNS server dynamically creates a corresponding DNS record, enabling workloads to discover each other using persistent domain names.

---

## 2. High-Level Architecture & Deployment Model

A frequent question is: _Is CoreDNS an operating system daemon (systemd service) running on each node, or is it a container?_

**CoreDNS runs as standard Kubernetes Pods directly inside the cluster.**

```text
┌────────────────────────────────────────────────────────┐
│                   kube-apiserver                       │
│  Notifies CoreDNS of Service / Pod / Endpoint changes  │
└───────────────────────────┬────────────────────────────┘
                            │ Watch API (real-time stream)
                            ▼
┌────────────────────────────────────────────────────────┐
│             CoreDNS Deployment (kube-system)           │
│             Replicas: 2 (for High Availability)        │
│                                                        │
│  Config: Mounted from ConfigMap "coredns"              │
│  Plugins: kubernetes, forward, cache, health, errors   │
└───────────────────────────┬────────────────────────────┘
                            │ Exposes Port 53 UDP/TCP
                            ▼
┌────────────────────────────────────────────────────────┐
│             kube-dns Service (ClusterIP)               │
│                     10.96.0.10                         │
└───────────────────────────▲────────────────────────────┘
                            │ Queries Port 53
                            │
┌───────────────────────────┴────────────────────────────┐
│                    Application Pod                     │
│  /etc/resolv.conf ──► nameserver 10.96.0.10            │
└────────────────────────────────────────────────────────┘
```

### The Three Structural Pillars:

1. **The Deployment (`coredns`):**
   - Lives in the **`kube-system`** namespace.
   - Runs **2 replicas** by default for redundancy and high availability across worker nodes:
     ```bash
     kubectl get pods -n kube-system -l k8s-app=kube-dns
     # coredns-768b85b76f-abcde   1/1   Running
     # coredns-768b85b76f-xyz12   1/1   Running
     ```
   - Runs the exact same `coredns` binary executable that we explored in the standalone CoreDNS lecture.

2. **The Front-End Service (`kube-dns`):**
   - Because the CoreDNS Pods themselves have dynamic IPs, a stable virtual front-end is required.
   - Kubernetes creates a `ClusterIP` Service named **`kube-dns`** (named for backward compatibility with pre-1.12 clusters):
     ```bash
     kubectl get svc -n kube-system kube-dns
     # NAME       TYPE        CLUSTER-IP   PORT(S)
     # kube-dns   ClusterIP   10.96.0.10   53/UDP, 53/TCP
     ```
   - This static IP (typically `10.96.0.10`) is the single DNS server IP given to all application Pods.

3. **The Configuration Map (ConfigMap `coredns`):**
   - CoreDNS requires a `Corefile`. Rather than baking it into the container image, Kubernetes stores it in a ConfigMap named `coredns` in `kube-system`.
   - The ConfigMap is mounted into the CoreDNS containers at `/etc/coredns/Corefile`.

---

## 3. How CoreDNS Knows About Kubernetes: The `kubernetes` Plugin

In a traditional DNS server (like BIND9), an administrator manually writes zone files (`db.example.com`). In Kubernetes, CoreDNS uses the **`kubernetes` plugin**:

1. When a CoreDNS pod starts, the `kubernetes` plugin establishes a persistent `Watch` connection to the **Kubernetes API Server (`kube-apiserver`)**.
2. Whenever a new `Service` is created (`kubectl create service...`):
   - The API Server streams the event to CoreDNS.
   - CoreDNS dynamically creates an in-memory `A` record mapping the service name to its `ClusterIP`:
     $$\text{web-service.default.svc.cluster.local} \longrightarrow \text{10.100.200.5}$$
3. Whenever a Service or Pod is deleted, CoreDNS immediately purges the record from memory.
4. **No flat files or zone databases on disk are ever modified.**

---

## 4. The Kubernetes `Corefile` Deep Dive

To inspect the live CoreDNS configuration:

```bash
kubectl get cm coredns -n kube-system -o yaml
```

### The Standard `Corefile` Block:

```corefile
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

### Directive & Plugin Breakdown:

| Plugin / Option                  | Operational Purpose                                                                                                                                                                                   |
| :------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`.:53`**                       | Binds the server to listen on port 53 (UDP & TCP) for all root domain zones.                                                                                                                          |
| **`errors`**                     | Writes standard query processing errors to `stderr`.                                                                                                                                                  |
| **`health` / `ready`**           | Exposes HTTP health check endpoints (`/health`, `/ready`) on port 8080 used by Kubernetes liveness and readiness probes.                                                                              |
| **`kubernetes cluster.local`**   | The core plugin that integrates CoreDNS with the Kubernetes API. Tells CoreDNS that all domain queries ending with `.cluster.local` belong to this cluster.                                           |
| **`pods insecure`**              | Controls how Pod A-records are generated. `insecure` allows resolving dashed-IP pod names (e.g., `10-244-1-5.default.pod.cluster.local`) without verifying if the caller matches the pod's namespace. |
| **`forward . /etc/resolv.conf`** | For any domain that does **not** end in `cluster.local` (e.g. `google.com`), CoreDNS proxies the query to upstream DNS servers defined in the host node's `/etc/resolv.conf`.                         |
| **`cache 30`**                   | Retains DNS query results in memory for up to 30 seconds to minimize load on the API server.                                                                                                          |
| **`loop`**                       | Detects and breaks infinite DNS forwarding loops (common when host `/etc/resolv.conf` points back to 127.0.0.1 / CoreDNS).                                                                            |
| **`reload`**                     | Watches the mounted `Corefile`. When the ConfigMap is edited, CoreDNS dynamically reloads its config without requiring a pod restart.                                                                 |
| **`loadbalance`**                | Randomizes the order of A/AAAA records returned in DNS responses (round-robin DNS load balancing).                                                                                                    |

---

## 5. The Missing Link: Who Connects the Pods to CoreDNS?

How does an individual application Pod know that `10.96.0.10` is its DNS server?

**The node's `kubelet` process is solely responsible.**

1. Every worker node runs `kubelet`.
2. When `kubelet` starts, it reads its configuration from `/var/lib/kubelet/config.yaml`:
   ```yaml
   clusterDNS:
     - 10.96.0.10
   clusterDomain: cluster.local
   ```
3. When `kubelet` provisions a new Pod on that node, it configures the container's network namespace and writes `/etc/resolv.conf` inside the Pod:
   ```ini
   nameserver 10.96.0.10
   search default.svc.cluster.local svc.cluster.local cluster.local
   options ndots:5
   ```

Because of this automated injection:

- All Pods point to the `kube-dns` Service IP (`10.96.0.10`).
- Short names like `web-service` automatically match the search domains and resolve without needing the full FQDN.

---

## 6. End-to-End DNS Query Walkthrough

Assume a Pod named `frontend` in namespace `prod` makes an HTTP request:

```bash
curl api-service
```

Here is the exact network path the query travels:

```text
1. Frontend Pod (Local Resolver):
   - "api-service" has 0 dots (< ndots:5).
   - Appends the 1st search path: "api-service.prod.svc.cluster.local".
   - Sends a UDP query packet to 10.96.0.10:53.

2. Node Linux Kernel (Netfilter / iptables):
   - Intercepts destination 10.96.0.10:53.
   - Applies DNAT: translates 10.96.0.10 to an active CoreDNS Pod IP (e.g., 10.244.0.4:53).

3. CoreDNS Pod:
   - Matches the zone ".cluster.local" -> hands query to the "kubernetes" plugin.
   - Queries its internal in-memory cache for Service "api-service" in namespace "prod".
   - Finds ClusterIP: 10.100.50.22.
   - Sends the DNS response packet back to the Frontend Pod.

4. Frontend Pod:
   - Receives IP 10.100.50.22 and initiates the TCP connection to the Service.
```

If the Pod had queried an external site (`curl google.com`):

- CoreDNS would note that `google.com` does not match `.cluster.local`.
- Via the `forward . /etc/resolv.conf` directive, CoreDNS would forward the request to the upstream DNS server (e.g. `8.8.8.8`) and relay the resolved public IP back to the Pod.

---

## 7. Verification and Diagnostic Commands

### 1. View CoreDNS ConfigMap & Corefile

```bash
kubectl describe cm coredns -n kube-system
```

### 2. View CoreDNS Deployment and Pods

```bash
kubectl get deployment coredns -n kube-system
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

### 3. Check the CoreDNS Service IP & Endpoints

```bash
kubectl get svc kube-dns -n kube-system
kubectl get endpoints kube-dns -n kube-system
```

### 4. Inspect Kubelet DNS Configuration on a Node

```bash
cat /var/lib/kubelet/config.yaml | grep -A 2 clusterDNS
```

### 5. Check CoreDNS Logs

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

---

## 8. Summary Comparison: Standalone CoreDNS vs. Kubernetes CoreDNS

| Feature           | Standalone CoreDNS                     | Kubernetes CoreDNS                            |
| :---------------- | :------------------------------------- | :-------------------------------------------- |
| **Execution**     | Binary running directly on Linux OS    | Pods inside `kube-system` Deployment          |
| **Configuration** | Flat file `./Corefile` on disk         | Kubernetes ConfigMap (`coredns`)              |
| **Backend Data**  | `/etc/hosts` or static zone files      | Dynamic stream from `kube-apiserver`          |
| **Service IP**    | Physical / Virtual host IP             | Stable virtual `ClusterIP` (`10.96.0.10`)     |
| **Client Config** | Manually written to `/etc/resolv.conf` | Automatically injected into Pods by `kubelet` |
