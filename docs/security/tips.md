# CKA Security & Certificate Exam Tips

A condensed checklist of high-yield tips and speed reflexes for security, TLS, and certificate scenarios in the CKA exam.

---

## 1. Quick Certificate Expiration & Renewal Reflexes

- **Check all expiration dates instantly:**
  ```bash
  kubeadm certs check-expiration
  ```
- **Renew all certificates on control plane:**
  ```bash
  kubeadm certs renew all
  ```
- **After renewing:** Remember to restart `kubelet` so static pods reload new certificates:
  ```bash
  systemctl restart kubelet
  ```

---

## 2. Certificate Inspection with OpenSSL

When an exam question asks for Common Name (CN), Issuer, validity dates, or Subject Alternative Names (SANs):

```bash
# Print certificate text
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

# Extract Common Name directly
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -subject

# Extract Issuer directly
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -issuer

# Check expiration date
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate
```

---

## 3. Fast CSR Generation Workflow

When asked to create a user with group membership:
- `CN` = Username
- `O` = Group name

```bash
# 1. Generate private key and CSR
openssl genrsa -out dev-user.key 2048
openssl req -new -key dev-user.key -subj "/CN=dev-user/O=developers" -out dev-user.csr

# 2. Extract single-line base64
export BASE64_CSR=$(cat dev-user.csr | base64 | tr -d '\n')

# 3. Apply CSR manifest (signerName: kubernetes.io/kube-apiserver-client)
kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: dev-user-csr
spec:
  request: ${BASE64_CSR}
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# 4. Approve CSR
kubectl certificate approve dev-user-csr

# 5. Extract signed certificate
kubectl get csr dev-user-csr -o jsonpath='{.status.certificate}' | base64 -d > dev-user.crt
```

---

## 4. `Kubeconfig` Configuration Without Manual Editing

Never manually edit YAML files when configuring kubeconfigs during the exam:

```bash
# Add credentials (client cert and key)
kubectl config set-credentials dev-user \
  --client-certificate=dev-user.crt \
  --client-key=dev-user.key \
  --embed-certs=true

# Add context
kubectl config set-context dev-user-ctx \
  --cluster=kubernetes \
  --user=dev-user \
  --namespace=development

# Switch context
kubectl config use-context dev-user-ctx
```

---

## 5. Testing Permissions Quickly (`kubectl auth can-i`)

```bash
# Can user perform action?
kubectl auth can-i create deployments --as=dev-user -n development

# Can service account perform action?
kubectl auth can-i get secrets --as=system:serviceaccount:default:my-sa

# List all permissions of a user
kubectl auth can-i --list --as=dev-user
```
