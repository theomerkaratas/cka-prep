# Control Plane Failure: Exam Workflow and Tips

## Exam Workflow

1. **Check the nodes:** `kubectl get nodes`. If `kubectl` fails, go straight to `crictl` and the manifests.
2. **Check the pods:** `kubectl get pods -n kube-system`.
3. **Find the broken component:** pod status, or `crictl ps -a` when the API is down.
4. **Read the logs:** `kubectl logs` for pods, `journalctl` for services.
5. **Fix the manifest or config**, then confirm the pod or service comes back.
6. **Verify:** `kubectl get nodes` and a test pod.

---

## Exam Tips

- **Use `crictl` when `kubectl` is down.** It talks to the container runtime directly.
- **Static pod manifests are the source of truth.** Edit the file in `/etc/kubernetes/manifests/`, not the pod.
- **Check paths and typos first.** Most control plane failures come from a wrong flag or path.
- **Use `--previous` for crashing pods.** The current container may have no useful logs yet.
- **Check the kubelet.** If it is down, no static pods run.

### References
- [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Troubleshooting with crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
