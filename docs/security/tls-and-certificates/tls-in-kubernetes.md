- **Overview:**
  - All communication within a Kubernetes cluster (via `kubectl`, internal components, or direct API calls) must be encrypted and secured over TLS.
  - Kubernetes uses **mutual TLS (mTLS)**: servers validate clients, and clients validate servers using respective certificate/key pairs.

---

### 1. Certificate Authority (CA)

- **Cluster Root of Trust:**
	- Kubernetes requires at least one CA to sign all component certificates.
	- You can use a single CA for the entire cluster or separate CAs (e.g., one for Kubernetes components and a dedicated one for `etcd`).
	- **Files:**
		- `ca.crt`: CA public certificate (distributed to all components to verify each other).
		- `ca.key`: CA private key (kept strictly secure to sign new certificates).

---

### 2. Server Certificates (For Components Exposing HTTPS Endpoints)

Components that listen for incoming HTTPS connections act as servers and require a server certificate and private key:

- **`kube-apiserver`:**
	- Exposes the central Kubernetes HTTPS API.
	- **Files:** `apiserver.crt`, `apiserver.key`
- **`etcd server`:**
	- Stores cluster state and exposes its own secure key-value store endpoint.
	- **Files:** `etcdserver.crt`, `etcdserver.key`
- **`kubelet server`:**
	- Runs on each worker node and exposes an HTTPS API endpoint for the `kube-apiserver` to monitor and manage pods/nodes.
	- **Files:** `kubelet.crt`, `kubelet.key`

---

### 3. Client Certificates (For Components Connecting to the API Server)

Components and users that initiate connections must authenticate themselves to the `kube-apiserver` using client certificates:

- **Cluster Administrators:**
	- Authenticate administrative operations (e.g., via `kubectl`).
	- **Files:** `admin.crt`, `admin.key`
- **Control Plane Clients:**
	- `kube-scheduler`: `scheduler.crt`, `scheduler.key`
	- `kube-controller-manager`: `controller-manager.crt`, `controller-manager.key`
- **Node Clients:**
	- `kube-proxy`: `kube-proxy.crt`, `kube-proxy.key`
	- `kubelet client`: `kubelet-client.crt`, `kubelet-client.key` (when registering/talking back to `kube-apiserver`)

---

### 4. Dual-Role Components (Server as a Client)

- Some components act as both servers and clients depending on who initiates the request:
	- **`kube-apiserver`:**
		- **Acts as a Server** when responding to clients (`admin`, `scheduler`, `controller-manager`).
		- **Acts as a Client** when querying the `etcd server` or sending requests down to node `kubelet` instances.
		- Can use dedicated client certificates (`apiserver-etcd-client.crt`, `apiserver-kubelet-client.crt`) or reuse its primary `apiserver.crt` and `apiserver.key` if configured.

---

## 5. Kubernetes PKI Architecture Summary Matrix

```text
                  +-----------------------------+
                  |         Kubernetes CA       |
                  |     (ca.crt / ca.key)       |
                  +--------------+--------------+
                                 |
         +-----------------------+-----------------------+
         | Signs Server Certs                            | Signs Client Certs
         v                                               v
+------------------+                           +--------------------+
|  kube-apiserver  | <--- HTTPS (Port 6443) -- |    kubectl admin   |
| (apiserver.crt)  | <--- HTTPS (Port 6443) -- |   kube-scheduler   |
+--------+---------+ <--- HTTPS (Port 6443) -- | controller-manager |
         |                                     |     kube-proxy     |
         | (Client)                            +--------------------+
         |
         +--- HTTPS (Port 10250) ------------> +--------------------+
         |    (apiserver-kubelet-client.crt)   |    kubelet server  |
         |                                     |    (kubelet.crt)   |
         |                                     +--------------------+
         | (Client)
         +--- HTTPS (Port 2379) -------------> +--------------------+
              (apiserver-etcd-client.crt)      |     etcd server    |
                                               | (etcd/server.crt)  |
                                               +--------------------+
```
