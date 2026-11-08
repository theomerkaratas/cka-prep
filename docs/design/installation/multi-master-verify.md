# Verify the HA Cluster (Multiple Masters)

```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
```

Check the following:

- All three masters and the worker are `Ready`.
- Each master runs its own `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, and `etcd` pod.
- The `etcd` pods are on the three masters, and each one is `Running`.

To test the load balancer, stop the API server on one master (or power that master off). `kubectl get nodes` should keep working through the load balancer.

## Where etcd runs

This setup uses **stacked** etcd: etcd runs on the same three masters as the control plane. It is the simpler option and needs only three servers. If one master fails, you lose one etcd member and one control plane instance together. The cluster keeps working because two of the three etcd members still form a quorum.

An **external** etcd cluster runs on its own servers, separate from the masters. It needs more machines and more setup, but a master failure does not touch etcd. See [etcd in HA](../ha/etcd-in-ha.md) for the trade-offs.
