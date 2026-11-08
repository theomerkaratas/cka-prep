# Control Plane Failure: Node and Pod Status

```bash
# List nodes with extra details
kubectl get nodes -o wide
# Show details and events for a node
kubectl describe node <node>
# List system pods with extra details
kubectl get pods -n kube-system -o wide
# Show scheduler pod details and events
kubectl describe pod kube-scheduler-master -n kube-system
```

## Healthy kube-system pods

A healthy kubeadm cluster shows all control plane pods `Running` with `READY` 1/1 (or 2/2 for network add-ons such as Weave Net) and few restarts:

```text
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-78fcdf6894-5dntv               1/1     Running   0          1h
etcd-master                            1/1     Running   0          1h
kube-apiserver-master                  1/1     Running   0          1h
kube-controller-manager-master         1/1     Running   0          1h
kube-proxy-fvbpj                       1/1     Running   0          1h
kube-scheduler-master                  1/1     Running   0          1h
weave-net-7kd52                        2/2     Running   1          1h
```

Next: [Service Status](service-status.md), [Logs](logs.md).
