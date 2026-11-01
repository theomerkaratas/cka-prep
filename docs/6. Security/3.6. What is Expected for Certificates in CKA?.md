In the Certified Kubernetes Administrator (CKA) exam, you are **not** expected to manually set up a CA or sign all cluster certificates from scratch using raw OpenSSL. Instead, the tasks focus on practical administration and troubleshooting across **4 main scenarios**:

---

### 1. Managing Users via Kubernetes CSR API (Most Common Task)

A typical task asks you to *"Create credentials for developer John, approve his CSR, and grant him access using RBAC."* In Kubernetes, this is handled through the native `CertificateSigningRequest` API, not by accessing the root CA directly.

- **Step 1: Generate Private Key and CSR locally:**
  ```bash
  openssl genrsa -out john.key 2048
  openssl req -new -key john.key -subj "/CN=john/O=developers" -out john.csr
  ```

- **Step 2: Base64-encode the CSR file:**
  ```bash
  cat john.csr | base64 | tr -d "\n"
  ```

- **Step 3: Create the CSR manifest (`csr.yaml`) and apply it:**
  ```yaml
  apiVersion: certificates.k8s.io/v1
  kind: CertificateSigningRequest
  metadata:
    name: john-csr
  spec:
    request: <INSERT_BASE64_OUTPUT_HERE>
    signerName: kubernetes.io/kube-apiserver-client
    usages:
    - client auth
  ```
  ```bash
  kubectl apply -f csr.yaml
  ```

- **Step 4: Approve the Certificate:**
  ```bash
  kubectl certificate approve john-csr
  ```

- **Step 5: Extract the signed certificate:**
  ```bash
  kubectl get csr john-csr -o jsonpath='{.status.certificate}' | base64 -d > john.crt
  ```

---

### 2. Checking and Renewing Expired Certificates with `kubeadm`

A scenario might state: *"Check cluster certificate expiration and renew any expired certificates on the control plane."*

- **Step 1: Check expiration status:**
  ```bash
  kubeadm certs check-expiration
  ```

- **Step 2: Renew all certificates:**
  ```bash
  kubeadm certs renew all
  ```

- **Step 3: Restart control plane components:**
  - Control plane static pods must restart to load renewed certificates.
  - Temporarily move manifests out and back into `/etc/kubernetes/manifests/`, or restart `kubelet`:
    ```bash
    systemctl restart kubelet
    ```

---

### 3. Cluster Troubleshooting (Broken Control Plane)

A broken cluster scenario where `kubectl get nodes` returns connection errors (e.g., `connection refused` or TLS handshake failure).

- **Common causes to check:**
  - **Wrong Certificate Paths:** Check `/etc/kubernetes/manifests/kube-apiserver.yaml` for typos in flags like `--tls-cert-file`, `--tls-private-key-file`, or `--client-ca-file`.
  - **Expired Certificates:** Inspect control plane container logs using `crictl logs <container-id>` to identify `"certificate has expired"` errors, then renew using `kubeadm certs renew`.
  - **Inspecting Certificate Details with OpenSSL:**
    ```bash
    openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
    ```
    *(Used to verify `Subject` (CN), `Validity` dates, and `X509v3 Subject Alternative Name` (SAN) entries).*

---

### 4. Configuring `kubeconfig` and Contexts

A task may ask you to add the newly generated client certificates into `kubeconfig` and configure a usable context.

- **Add client credentials:**
  ```bash
  kubectl config set-credentials john \
    --client-certificate=john.crt \
    --client-key=john.key \
    --embed-certs=true
  ```

- **Set up the context:**
  ```bash
  kubectl config set-context john-context \
    --cluster=kubernetes \
    --user=john
  ```

- **Switch and test the context:**
  ```bash
  kubectl config use-context john-context
  ```

---

### Key Takeaway for CKA

- Do **not** spend time memorizing full OpenSSL CA generation scripts.
- Focus on mastering:
  - `kubectl certificate approve / deny`
  - `kubeadm certs check-expiration / renew`
  - `kubectl config set-credentials / set-context`
  - Inspecting static pod manifests under `/etc/kubernetes/manifests/`