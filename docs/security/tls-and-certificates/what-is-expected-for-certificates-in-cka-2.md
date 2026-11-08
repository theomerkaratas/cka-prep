You will not be asked theoretical questions; instead, you will be given a **broken cluster** and asked to fix it.

Here are the typical real exam scenarios:

---

### Scenario 1: "The API Server is down, find the issue and fix it" (Most Common)

- **The Symptom:**
  - You switch to the target node and run `kubectl get nodes`, receiving an error like:
    ```text
    The connection to the server <ip>:6443 was refused - did you specify the right host or port?
    ```

- **Expected Troubleshooting Steps:**
  1. Identify crashed control plane containers using `crictl`:
     ```bash
     crictl ps -a | grep apiserver
     ```
  2. Inspect the failed container logs:
     ```bash
     crictl logs <container-id>
     ```
     You will often spot errors like:
     `stat /etc/kubernetes/pki/apiserver-etcd.crt: no such file or directory` (or a typo in a certificate file path).
  3. Edit the static pod manifest:
     ```bash
     vim /etc/kubernetes/manifests/kube-apiserver.yaml
     ```
  4. Fix the misspelled certificate parameter (e.g., correct `--etcd-certfile` or `--tls-cert-file`).
  5. Save and exit; `kubelet` automatically detects changes and restarts the static pod.

---

### Scenario 2: "Check Certificate Details (CN, Expiration, SANs)"

- **The Task:**
  - The question provides a path to a certificate file and asks you to find its Common Name (CN), Issuer, or Expiration date:
    *"Inspect `/etc/kubernetes/pki/apiserver.crt` and determine who issued it and when it expires."*

- **Command to Run:**
  ```bash
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
  ```
  Look for:
  - `Subject: CN = ...`
  - `Issuer: CN = ...`
  - `Validity -> Not After: ...`
  - `X509v3 Subject Alternative Name: ...`

---

### Scenario 3: "Certificates Expired, Renew Them"

- **The Task:**
  - Cluster communication fails due to expired control plane certificates, or the task explicitly asks:
    *"Renew all expired control plane certificates on controlplane node."*

- **Commands to Run:**
  1. Check expiration dates across all kubeadm-managed certificates:
     ```bash
     kubeadm certs check-expiration
     ```
  2. Renew all certificates:
     ```bash
     kubeadm certs renew all
     ```
  3. Restart the `kubelet` service so the static pods pick up the new certificates:
     ```bash
     systemctl restart kubelet
     ```

---

### Key Exam Takeaways & Reflexes

| Situation | First Action |
| :--- | :--- |
| `kubectl` fails with `connection refused` | Run `crictl ps -a` & `crictl logs <id>` to inspect failing static pods. |
| Verify configured cert paths | Check manifests in `/etc/kubernetes/manifests/`. |
| Inspect a specific certificate file | Run `openssl x509 -in <path> -text -noout`. |
| Bulk expiration check | Run `kubeadm certs check-expiration`. |