# Control Plane Failure: Service Status (Non-kubeadm)

If the components run as systemd services, check each one. `systemctl status <service>` gives the same result.

```bash
# Check API server service status
service kube-apiserver status
# Check controller manager service status
service kube-controller-manager status
# Check scheduler service status
service kube-scheduler status
# Check kubelet service status
service kubelet status
# Check kube-proxy service status
service kube-proxy status
```

A healthy service shows `Active: active (running)`. If not, check its logs with `journalctl` (see [Logs](logs.md)).
