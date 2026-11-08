# Control Plane Failure: Logs

## kubeadm (static pods)

```bash
# View API server logs
kubectl logs kube-apiserver-master -n kube-system
# View the scheduler's previous container logs
kubectl logs kube-scheduler-master -n kube-system --previous
# View controller manager logs
kubectl logs kube-controller-manager-master -n kube-system
```

## systemd services (non-kubeadm)

```bash
# Show the last 50 API server log lines
journalctl -u kube-apiserver --no-pager | tail -50
# Follow kubelet logs in real time
journalctl -u kubelet -f
```
