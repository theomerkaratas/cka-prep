# Network Failure: Exam Workflow and Checklist

## Order to Check

| # | Layer | Quick command | Healthy looks like |
| :-: | :--- | :--- | :--- |
| 1 | Pods | `kubectl get pods -l app=hostnames` | `Running`, `1/1`, few restarts |
| 2 | Pod IP | `wget -qO- http://<pod-ip>:<port>` from a test pod | App answers |
| 3 | Service | `kubectl get endpointslices -l kubernetes.io/service-name=hostnames` | Pod IPs listed, not `<none>` |
| 4 | CoreDNS | `kubectl get pods -n kube-system -l k8s-app=kube-dns` | `Running` |
| 5 | DNS names | `nslookup hostnames.default.svc.cluster.local` | Returns the Service IP |
| 6 | CNI | `kubectl get pods -n kube-flannel` or `-n calico-system` | `Running`, clean logs |
| 7 | kube-proxy | `kubectl get pods -n kube-system -l k8s-app=kube-proxy` | `Running` on every node |

## Common Causes

- **Pods not Ready:** the app or its probe is failing. Check `describe` and logs.
- **Endpoints `<none>`:** the Service selector does not match the pod labels.
- **Pods answer by IP but not by Service:** wrong `port` or `targetPort`, or kube-proxy rules are missing.
- **CoreDNS `Pending`:** the CNI plugin is not working.
- **CoreDNS `CrashLoopBackOff`:** read the CoreDNS logs and check the Corefile.
- **Pods cannot talk at all:** check the CNI plugin, and check that `/etc/cni/net.d/` has the config you expect.

## Key Points

- Start with the pods, then the Service, then DNS, then the network layers.
- Stop at the first layer that fails and fix it before going further.
- Use `kubectl describe`, `kubectl logs`, and `kubectl get endpointslices` at every step.
