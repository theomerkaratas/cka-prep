# Worker Plane Failure: Exam Workflow and Tips

## Exam Workflow

1. **Check the nodes:** `kubectl get nodes`. Find the `NotReady` or `Unknown` node.
2. **Describe the node:** `kubectl describe node <node>` and read the conditions and heartbeat time.
3. **Log in to the node:** check whether it is online or crashed.
4. **Check resources:** `top`, `free -h`, `df -h`.
5. **Check the kubelet:** `service kubelet status`, then `journalctl -u kubelet`.
6. **Check the kubelet certificates:** expiry, group, and CA.
7. **Fix and verify:** `kubectl get nodes` should show the node `Ready`.

---

## Exam Tips

- **Start with `describe node`.** The conditions point to the cause.
- **`Unknown` usually means the node is down or unreachable.** Check the machine first.
- **Check the kubelet first when the node is up.** Most `NotReady` nodes are a stopped or misconfigured kubelet.
- **Expired kubelet certificates** stop the node from joining, so check the dates.

### References
- [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
