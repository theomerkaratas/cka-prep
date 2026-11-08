## 1. Overview & Problem Statement

All communication between Kubernetes components (API server, kubelet, etcd, controller manager, scheduler) and external users is secured using TLS certificates.

- **The Problem:** 
  When a new user (e.g., `jane`) joins the team, she needs cluster access via a certificate.
  - In a traditional/manual setup: An admin has to SSH into the master node, access the root CA key pair (`ca.key` and `ca.crt`), and manually sign the user's Certificate Signing Request (CSR) using OpenSSL tools.
  - **Risk & Inefficiency:** Logging into the master node exposes the private CA key (`ca.key`). If this key is compromised, the entire cluster security is compromised. Manually signing certificates for growing teams and rotating them when expired is tedious and non-scalable.

- **The Solution:** 
  Kubernetes provides a built-in **Certificates API** (`certificates.k8s.io/v1`). It allows users and components to request certificate signing directly via Kubernetes API objects, eliminating the need for manual access to the master node's CA private key.

---

## 2. Step-by-Step Workflow

### Step 1: Generate Private Key and CSR (Client Side)

The user generates their private key and a CSR locally:

```bash
# Generate private key
openssl genrsa -out jane.key 2048

# Generate CSR
# CN (Common Name) becomes the Kubernetes username: "jane"
# O (Organization) maps to Kubernetes groups (e.g., "developers")
openssl req -new -key jane.key -subj "/CN=jane/O=developers" -out jane.csr
```
### Step 2: Encode the CSR and Create the Manifest

Encode the CSR file content in base64 as a single line (strip newlines):

```sh
cat jane.csr | base64 | tr -d '\n'
# or
cat jane.csr | base64 -w 0
# or
export BASE64_CSR=$(cat jane.csr | base64 | tr -d '\n')
```
Create the `CertificateSigningRequest` manifest:
```yaml
# jane-csr.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane-csr
spec:
  groups:
  - system:authenticated
  request: <BASE64_ENCODED_CSR_STRING_HERE>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # Optional (e.g., 24 hours)
  usages:
  - client auth
```
> **Important Notes on Manifest Fields:**
> 
> - `spec.signerName`: Tells Kubernetes which signer to use. For user client certificates accessing the API server, use `kubernetes.io/kube-apiserver-client`.
> - `spec.usages`: Must include `client auth` for user authentication.

Apply the manifest to the cluster:
```sh
kubectl apply -f jane-csr.yaml
```
### Step 3: Inspect and Approve the CSR

List all pending CSRs:
```sh
kubectl get csr
```
Approve or deny the CSR:
```sh
# Approve
kubectl certificate approve jane-csr

# Deny (if needed)
kubectl certificate deny jane-csr
```
Verify that the condition changes to `Approved,Issued`:
```sh
kubectl get csr jane-csr
```
-----------

### Step 4: Extract the Signed Certificate

Once approved, the controller populates the `status.certificate` field. Extract and decode it to a file:

```sh
kubectl get csr jane-csr -o \
  jsonpath='{.status.certificate}' | base64 -d > jane.crt

# Inspect the generated certificate:
openssl x509 -in jane.crt -text -noout
```
The user `jane` can now configure her `kubeconfig` using:

- `jane.key` (Private key)
- `jane.crt` (Signed client certificate)
- `ca.crt` (Cluster CA certificate)

---

## 3. Control Plane Internals: Who Signs the Certificate?

Certificate signing operations in Kubernetes are handled by the **`kube-controller-manager`**.

Inside `kube-controller-manager`, two internal controllers manage this lifecycle:

1. **CSR Approving Controller**: Validates permissions to approve requests.
2. **CSR Signing Controller**: Signs approved requests using the cluster CA key and certificate.

You can verify the CA file paths in the static pod manifest:

```sh
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
```

Key flags configured in the manifest:

```yaml
spec:
  containers:
  - command:
    - kube-controller-manager
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    ...
```

## 4. Key CKA Exam Takeaways & Tips

1. **JSONPath Extraction:**  
    Remember the command to extract the certificate:
2. **Signer Names:**  
    Always use the appropriate signer name (most common for users: `kubernetes.io/kube-apiserver-client`).
3. **No Status/Conditions on Creation:**  
    Never put `status` or `conditions` under the manifest when submitting a new CSR.
4. **Base64 Encoding:**  
    Always strip newline characters (`tr -d '\n'`) when inserting the encoded CSR into `spec.request`.

```sh
kubectl get csr <csr-name> -o \
  jsonpath='{.status.certificate}' | base64 -d > <output-file>.crt
```


### 🛠️ Step-by-Step Commands for the CKA Exam

This type of question is frequently asked in the exam (e.g., *"Create a CSR for user X, approve it, and extract the certificate"*).

#### Step 1: Generate Key and CSR (Client Side)
```bash
# Generate private key
openssl genrsa -out jane.key 2048

# Generate CSR (CN = Username, O = Group)
# For example, if Jane should belong to the 'developers' group: /CN=jane/O=developers
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
```
#### Step 2: Prepare and Apply the CSR YAML

```sh
# Encode CSR content to base64 (single line)
export BASE64_CSR=$(cat jane.csr | base64 | tr -d '\n')
```
Here is the correct YAML manifest:

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane-csr
spec:
  request: <BASE64_CSR_HERE>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # optional: 1 day
  usages:
  - client auth
```

Apply it to the cluster:

```sh
kubectl apply -f jane-csr.yaml
```
#### Step 3: Check Status and Approve

```sh
# View pending requests
kubectl get csr
# In the output, you will see REQUESTOR, AGE, and CONDITION: Pending

# Approve the request
kubectl certificate approve jane-csr
```
#### Step 4: Extract the Signed Certificate

In the exam, they usually ask: _"Save the certificate to /root/jane.crt"_:

```sh
kubectl get csr jane-csr -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
```