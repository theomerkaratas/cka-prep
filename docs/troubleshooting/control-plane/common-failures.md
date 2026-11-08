# Control Plane Failure: Common Failures and Fixes

### API server crashes on start
- **Cause:** Typo in a flag, wrong cert or key path, or wrong etcd endpoint in `kube-apiserver.yaml`.
- **Check:** `crictl logs` or `kubectl logs kube-apiserver-master -n kube-system`.
- **Fix:** Correct the value in `/etc/kubernetes/manifests/kube-apiserver.yaml`.

### Scheduler not running (pods stay Pending)
- **Cause:** Scheduler manifest is broken, or its kubeconfig path is wrong.
- **Check:** `kubectl get pods -n kube-system` and the scheduler logs.
- **Fix:** Correct `/etc/kubernetes/manifests/kube-scheduler.yaml`, such as the `--kubeconfig` path.

### Controller manager not running (no pods created for Deployments)
- **Cause:** Same pattern: broken manifest or kubeconfig, or missing cert files.
- **Check:** `kubectl logs kube-controller-manager-master -n kube-system`.
- **Fix:** Correct `/etc/kubernetes/manifests/kube-controller-manager.yaml`.

### etcd unreachable
- **Cause:** Wrong `--etcd-servers` in the API server, or wrong etcd certs.
- **Check:** Run the `etcdctl endpoint health` command in [Certs and etcd](certs-and-etcd.md); check `--etcd-servers` in `kube-apiserver.yaml`.
- **Fix:** Correct the endpoint or cert paths.

### Node NotReady
- **Cause:** Kubelet is stopped or misconfigured.
- **Check:** `service kubelet status`, then `journalctl -u kubelet`.
- **Fix:** `systemctl restart kubelet` after fixing the cause; check swap, cgroup driver, and the kubelet config path.

### Expired certificates
- **Cause:** Control plane certs have expired.
- **Check:** `kubeadm certs check-expiration`.
- **Fix:** `kubeadm certs renew all`, then restart the control plane pods (or the kubelet on the node).
