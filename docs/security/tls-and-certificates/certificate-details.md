# Inspecting Kubernetes Certificates (`kubeadm`)

To perform a health check on cluster certificates, start by identifying the file locations from the static pod manifests and then inspect the certificate details using OpenSSL.

### 1. Verification Commands

- **Find configured certificate paths:**
  ```bash
  cat /etc/kubernetes/manifests/kube-apiserver.yaml
  ```
- **Inspect certificate details (CN, Issuer, Validity, SANs):**
  ```bash
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
  ```
- **Quick expiration overview across all `kubeadm` certs:**
  ```bash
  kubeadm certs check-expiration
  ```

---

## Kubernetes PKI Certificate Map

### 1. Kubernetes Root CA (`/etc/kubernetes/pki/`)
- **`ca.crt` (Root CA Certificate):**
  - **Type:** CA Certificate
  - **CN / Issuer:** `kubernetes`
  - **Purpose:** Verifies signatures for all control plane and client certificates.
- **`ca.key` (Root CA Private Key):**
  - **Type:** CA Key
  - **Purpose:** Signs new certificates issued by the Kubernetes CA.

---

### 2. `kube-apiserver` Certificates (`/etc/kubernetes/pki/`)

#### A. Server Role (Incoming connections)
- **`apiserver.crt` & `apiserver.key`:**
  - **Type:** Server Certificate / Key
  - **CN:** `kube-apiserver`
  - **Issuer:** `kubernetes`
  - **Subject Alternative Names (SANs):**
    - `DNS:master` (or `controlplane` hostname)
    - `DNS:kubernetes`
    - `DNS:kubernetes.default`
    - `DNS:kubernetes.default.svc`
    - `DNS:kubernetes.default.svc.cluster.local`
    - `IP:10.96.0.1` (Cluster default Service IP)
    - `IP:<Node-Internal-IP>` (e.g., `172.17.0.27`)
- **`ca.crt`:**
  - Configured via `--client-ca-file` to authenticate connecting clients (users, controllers, schedulers).

#### B. Client Role (Outgoing connections)
- **`apiserver-kubelet-client.crt` & `.key`:**
  - **Type:** Client Certificate / Key
  - **CN:** `kube-apiserver-kubelet-client`
  - **Organization (Group):** `system:masters`
  - **Purpose:** Used by the API server to authenticate when calling the `kubelet` API (for `logs`, `exec`, metrics).
- **`apiserver-etcd-client.crt` & `.key`:**
  - **Type:** Client Certificate / Key
  - **CN:** `kube-apiserver-etcd-client`
  - **Issuer:** `etcd-ca`
  - **Purpose:** Used by the API server to query the `etcd` database.

---

### 3. `etcd` Certificates (`/etc/kubernetes/pki/etcd/`)
*`etcd` has its own dedicated CA distinct from the main Kubernetes CA.*

- **`etcd/ca.crt` & `etcd/ca.key`:**
  - **Type:** Dedicated CA Certificate / Key
  - **CN / Issuer:** `etcd-ca`
- **`etcd/server.crt` & `etcd/server.key`:**
  - **Type:** Server Certificate / Key
  - **CN:** Hostname of control plane (e.g., `controlplane` or `master`)
  - **Purpose:** Secures port `2379` for API server queries.
- **`etcd/peer.crt` & `etcd/peer.key`:**
  - **Type:** Peer Certificate / Key
  - **Purpose:** Secures port `2380` for cluster sync between `etcd` members.

---

### 4. `kubelet` Certificates (`/var/lib/kubelet/pki/`)

- **Server Certificate (`kubelet.crt` / `kubelet.key`):**
  - **Type:** Server Certificate / Key
  - **Purpose:** Secures HTTPS endpoint on port `10250` for incoming calls from the API server.
- **Client Certificate (`kubelet-client-current.pem`):**
  - **Type:** Client Certificate
  - **CN:** `system:node:<node-name>` (e.g., `system:node:node01`)
  - **Organization (Group):** `system:nodes`
  - **Purpose:** Used by the node's `kubelet` to authenticate against the `kube-apiserver`.

---

## Troubleshooting Certificate Issues & Fetching Logs

When certificate errors occur, core components fail to start. Depending on the failure layer, check logs in this order:

### Scenario A: API Server is Running
Use standard `kubectl` commands:
```bash
kubectl logs -n kube-system etcd-<master-node-name>
kubectl logs -n kube-system kube-apiserver-<master-node-name>
```

### Scenario B: API Server or etcd is Down (Kubectl Fails)
If control plane pods fail TLS handshakes, `kubectl` cannot talk to the cluster. Inspect container runtime directly with `crictl`:
```bash
# 1. List all running and stopped containers
crictl ps -a

# 2. View logs directly from the failing container
crictl logs <container-id>
```

### Scenario C: Hard Way Installation (Systemd Services)
If Kubernetes or `etcd` is installed natively as a system service rather than static pods:
```bash
journalctl -u etcd.service -l --no-pager
journalctl -u kube-apiserver.service -l --no-pager
```