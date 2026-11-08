# Verify the Cluster (Single Master)

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

All nodes should be `Ready`, and the control plane and add-on pods should be `Running`.

If a node stays `NotReady`, check that the CNI add-on is running in `kube-system`. If the master shows as tainted and you have no workers, see [Join the worker nodes](join-workers.md#taint-on-a-single-node-cluster).
