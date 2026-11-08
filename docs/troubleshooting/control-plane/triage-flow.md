# Control Plane Failure: Triage Flow

Check in this order. Each step narrows down where the failure is.

| # | Check | Command | Typical failure |
| :-: | :--- | :--- | :--- |
| 1 | Are all nodes healthy? | `kubectl get nodes` and `kubectl describe node <node>` | Node `NotReady`, kubelet down |
| 2 | Are control plane pods running? | `kubectl get pods -n kube-system` | CrashLoopBackOff or missing static pod |
| 3 | Is the API server healthy? | `kubectl get --raw='/readyz?verbose'` | Connection refused, cert errors |
| 4 | Are the services up (non-kubeadm)? | `service kube-apiserver status`, `service kube-controller-manager status`, `service kube-scheduler status` | Service stopped or failed |
| 5 | What do the logs say? | `kubectl logs kube-apiserver-master -n kube-system` or `journalctl -u kube-apiserver` | Bad flag, cert path, etcd unreachable |
| 6 | Are the node services running? | `service kubelet status` and `service kube-proxy status` | Kubelet or kube-proxy down, bad config, wrong cgroup driver |

The exact commands are in [Node and Pod Status](node-and-pod-status.md), [Service Status](service-status.md), [Logs](logs.md), [API Down with crictl](api-down-with-crictl.md), and [Certs and etcd](certs-and-etcd.md).
