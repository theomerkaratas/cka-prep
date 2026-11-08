# Certificate Creation in Kubernetes

- **Available Tools:**
  - Common tools for generating TLS certificates include `Easy-RSA`, `OpenSSL`, and `cfssl`.

---

### 1. Certificate Authority (CA) Setup

- **Overview:**
  - The CA is the root of trust in the cluster.
  - It signs certificates for all other Kubernetes components, users, and nodes.

- **Step 1: Generate the CA Private Key:**
  - Generate a secure 2048-bit private key:
    ```bash
    openssl genrsa -out ca.key 2048
    ```
  - **Output:** `ca.key` (must be kept secret and protected on the control plane).

- **Step 2: Generate the Certificate Signing Request (CSR):**
  - A CSR contains identity details (such as the Common Name) before an official signature is applied.
  - Set the Common Name (`CN`) to identify the CA:
    ```bash
    openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
    ```
  - **Output:** `ca.csr`

- **Step 3: Self-Sign the CA Certificate:**
  - Because this is the root authority, it signs its own certificate using its own private key:
    ```bash
    openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
    ```
  - **Output:** `ca.crt` (the public CA root certificate shared across all cluster components to verify signatures).

- **Result:**
  - The CA now has its key pair: `ca.key` (to sign future certificates) and `ca.crt` (to validate signed certificates).

### 2. Client Certificates

- **Overview:**
  - Client components (administrators, controllers, schedulers, proxies) use client certificates to authenticate themselves against the `kube-apiserver`.
  - The API server checks the Common Name (`CN`) for the user identity and the Organizational Unit (`OU`) for the group membership.

---

#### A. Generating Admin User Certificate

- **Step 1: Generate Private Key:**
  ```bash
  openssl genrsa -out admin.key 2048
  ```
  - **Output:** `admin.key`

- **Step 2: Generate Certificate Signing Request (CSR):**
  - Use `CN=kube-admin` for the username.
  - Set `OU=system:masters` to grant full cluster-admin privileges (a built-in Kubernetes superuser group):
    ```bash
    openssl req -new -key admin.key -subj "/CN=kube-admin/OU=system:masters" -out admin.csr
    ```
  - **Output:** `admin.csr`

- **Step 3: Sign with Cluster CA:**
  - Sign the CSR using the cluster's CA credentials (`ca.crt` and `ca.key`):
    ```bash
    openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out admin.crt
    ```
  - **Output:** `admin.crt`

---

#### B. System Component Client Certificates

- System components in the control plane follow the exact same process, but their `CN` must follow standard naming conventions prefixed with `system:`:
  - `kube-scheduler`: `/CN=system:kube-scheduler`
  - `kube-controller-manager`: `/CN=system:kube-controller-manager`
  - `kube-proxy`: `/CN=system:node-proxier` (or `system:kube-proxy`)

---

#### C. Authenticating with the API Server

- **Direct API Call via `curl`:**
  - Pass the client certificate, client private key, and the cluster CA certificate:
    ```bash
    curl https://kube-apiserver:6443/api/v1/pods \
      --key admin.key \
      --cert admin.crt \
      --cacert ca.crt
    ```

- **Using a `kubeconfig` File (Recommended):**
  - Instead of passing flags manually, store certificate and key paths in a configuration file:
    ```yaml
    apiVersion: v1
    kind: Config
    clusters:
    - cluster:
        certificate-authority: ca.crt
        server: https://kube-apiserver:6443
      name: kubernetes
    users:
    - name: kubernetes-admin
      user:
        client-certificate: admin.crt
        client-key: admin.key
    contexts:
    - context:
        cluster: kubernetes
        user: kubernetes-admin
      name: kubernetes-admin@kubernetes
    current-context: kubernetes-admin@kubernetes
    ```

---

#### D. Root CA Requirement

- **Bidirectional Trust (Mutual TLS):**
  - The client needs `ca.crt` to verify that the `kube-apiserver` is legitimate.
  - The server needs `ca.crt` to verify that the client certificate presented was signed by a trusted authority.
  - Every client and server configuration in Kubernetes must reference the CA root certificate for trust verification.
## Server Certificates

Components that listen for incoming HTTPS requests require server certificates to secure endpoints and authenticate to callers.
#### A. `etcd` Server Certificates
- **Overview:**
  - `etcd` requires server certificates for client connections (`kube-apiserver`) as well as **peer certificates** for cluster members talking to each other.
  - Requires the CA root certificate to validate clients and peers.

![[Screenshot 2026-10-25 at 1.58.03 PM.png|414]]

- **Configuration in `etcd.yaml` (Static Pod or Systemd):**
  ```yaml
  - etcd
    # Client communication (API Server -> etcd)
    - --listen-client-urls=https://127.0.0.1:2379
    - --advertise-client-urls=https://127.0.0.1:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --client-cert-auth=true
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt

    # Peer communication (etcd node <-> etcd node)
    - --listen-peer-urls=https://127.0.0.1:2380
    - --initial-advertise-peer-urls=https://127.0.0.1:2380
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-client-cert-auth=true
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
  ```

---

#### B. `kube-apiserver` Certificate (with Subject Alternative Names - SAN)

- **The Challenge:**
  - The API server is contacted using multiple hostnames and IP addresses:
    - Hostname / localhost: `kubernetes`, `127.0.0.1`
    - Cluster DNS names: `kubernetes.default`, `kubernetes.default.svc`, `kubernetes.default.svc.cluster.local`
    - Service Cluster IP: (e.g., `10.96.0.1`)
    - Node IP: (e.g., `172.17.0.87`)
  - A basic certificate will fail TLS verification if the target host does not match the certificate's `CN`. Therefore, all aliases must be included via **Subject Alternative Names (SAN)**.

- **Step 1: Generate Private Key:**
  ```bash
  openssl genrsa -out apiserver.key 2048
  ```

- **Step 2: Prepare OpenSSL Config (`openssl.cnf`):**
  ```ini
  [req]
  req_extensions = v3_req
  distinguished_name = req_distinguished_name
  [req_distinguished_name]
  [v3_req]
  basicConstraints = CA:FALSE
  keyUsage = nonRepudiation, digitalSignature, keyEncipherment
  subjectAltName = @alt_names
  [alt_names]
  DNS.1 = kubernetes
  DNS.2 = kubernetes.default
  DNS.3 = kubernetes.default.svc
  DNS.4 = kubernetes.default.svc.cluster.local
  IP.1 = 10.96.0.1
  IP.2 = 172.17.0.87
  IP.3 = 127.0.0.1
  ```

- **Step 3: Generate CSR with Config:**
  ```bash
  openssl req -new -key apiserver.key \
    -subj "/CN=kube-apiserver" \
    -out apiserver.csr \
    -config openssl.cnf
  ```

- **Step 4: Sign Certificate with SAN Extensions:**
  ```bash
  openssl x509 -req -in apiserver.csr \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out apiserver.crt \
    -extensions v3_req -extfile openssl.cnf \
    -days 365
  ```

- **`kube-apiserver` Startup Flags Breakdown:**
  - **Server Identity:**
    - `--tls-cert-file=/var/lib/kubernetes/apiserver.crt`
    - `--tls-private-key-file=/var/lib/kubernetes/apiserver.key`
  - **Client Verification (Incoming):**
    - `--client-ca-file=/var/lib/kubernetes/ca.pem` (validates client certs connecting to API server)
  - **Outgoing to `etcd` (as a Client):**
    - `--etcd-cafile=/var/lib/kubernetes/etcd-ca.pem`
    - `--etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt`
    - `--etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key`
  - **Outgoing to `kubelet` (as a Client):**
    - `--kubelet-certificate-authority=/var/lib/kubernetes/ca.pem`
    - `--kubelet-client-certificate=/var/lib/kubernetes/apiserver-kubelet-client.crt`
    - `--kubelet-client-key=/var/lib/kubernetes/apiserver-kubelet-client.key`

---

#### C. `kubelet` Server & Client Certificates

Each worker node runs a `kubelet` service that needs two sets of certificates:

1. **`kubelet` Server Certificate (Incoming from API Server):**
   - Secures the HTTPS endpoint on port `10250` used by `kube-apiserver` (for `kubectl logs`, `exec`, metrics).
   - Configured in `kubelet-config.yaml`:
     ```yaml
     apiVersion: kubelet.config.k8s.io/v1beta1
     kind: KubeletConfiguration
     authentication:
       x509:
         clientCAFile: "/var/lib/kubernetes/ca.pem" # Trust incoming requests from API server
     tlsCertFile: "/var/lib/kubelet/kubelet-node01.crt"
     tlsPrivateKeyFile: "/var/lib/kubelet/kubelet-node01.key"
     ```

2. **`kubelet` Client Certificate (Outgoing to API Server):**
   - The node needs to register itself and report pod statuses to the `kube-apiserver`.
   - **Naming Convention for Authorization (Node Authorizer & RBAC):**
     - Common Name format: `CN=system:node:<node-name>` (e.g., `/CN=system:node:node01`)
     - Organization format: `OU=system:nodes`
   - **Generation Steps:**
     ```bash
     openssl genrsa -out kubelet-node01-client.key 2048

     openssl req -new -key kubelet-node01-client.key \
       -subj "/CN=system:node:node01/O=system:nodes" \
       -out kubelet-node01-client.csr

     openssl x509 -req -in kubelet-node01-client.csr \
       -CA ca.crt -CAkey ca.key -CAcreateserial \
       -out kubelet-node01-client.crt -days 365
     ```
   - These client credentials are embedded into the node's `kubelet.kubeconfig` file so the node can authenticate continuously against the control plane.

