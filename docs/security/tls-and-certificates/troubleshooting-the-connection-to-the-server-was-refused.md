This error indicates that the `kube-apiserver` is either down, crashed in a loop, or unable to bind to port `6443`. Because `kubectl` relies entirely on the API server, you must troubleshoot at the node and container runtime layer.

---

### Step 1: Check the `kubelet` Service Status
The `kubelet` is responsible for launching and managing static control plane pods (`kube-apiserver`, `etcd`, etc.). If `kubelet` is down, nothing runs.

```bash
systemctl status kubelet
```

- **If inactive or failed:**
  ```bash
  systemctl restart kubelet
  journalctl -u kubelet -e --no-pager
  ```
- **If active (running):** Proceed to Step 2.

---

### Step 2: Inspect Containers Using `crictl`
When `kubectl` cannot connect, drop down to the container runtime CLI to check container state:

```bash
crictl ps -a | grep kube-apiserver
```

- **Scenario A: Container is repeatedly exiting (CrashLoop):**
  Inspect the exit logs from the latest failed container:
  ```bash
  crictl logs <container-id>
  ```
  *Common failure indicators in logs:*
  - `stat /etc/kubernetes/pki/...: no such file or directory` (Typo in certificate path)
  - `certificate has expired or is not yet valid` (Expired certs)
  - `context deadline exceeded` or connection refused to `127.0.0.1:2379` (`etcd` is down or unreachable)

- **Scenario B: No `kube-apiserver` container exists at all:**
  There is likely an invalid YAML syntax or indentation error in the static pod manifest preventing `kubelet` from parsing it.

---

### Step 3: Inspect and Fix Static Pod Manifests
Examine the API server static pod specification:

```bash
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Common issues to check:**
1. **Misspelled certificate/key flags or paths:**
   - `--tls-cert-file`
   - `--tls-private-key-file`
   - `--client-ca-file`
   - `--etcd-cafile`, `--etcd-certfile`, `--etcd-keyfile`
   - Cross-check against actual files on disk: `ls -la /etc/kubernetes/pki/`
2. **Incorrect etcd server endpoint:**
   - `--etcd-servers=https://127.0.0.1:2379`
3. **YAML syntax errors:**
   - Check bad indentation, missing hyphens in command args, or misplaced quotes.

> **Note:** Once you save and close the file, `kubelet` automatically detects the change within a few seconds and attempts to recreate the pod.

---

### Step 4: Verify `etcd` Health (If API Server Complains About etcd)
If the API server logs show failures contacting the database, check `etcd`:

```bash
crictl ps -a | grep etcd
crictl logs <etcd-container-id>
```

If needed, inspect its manifest as well:
```bash
vim /etc/kubernetes/manifests/etcd.yaml
```

---

### Step 5: Verify the Port and Re-test `kubectl`
Check if the port is actively listening again:

```bash
# Using ss or netstat
ss -tulpn | grep 6443
# or
netstat -tulpn | grep 6443
```

Once port `6443` is listening, re-run the cluster query:

```bash
kubectl get nodes
kubectl get po -A
```
