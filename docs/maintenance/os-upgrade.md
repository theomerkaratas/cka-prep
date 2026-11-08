# OS Upgrade

## Node Downtime and Pod Eviction

- Nodes are sometimes taken down on purpose, for example to upgrade base software or apply patches.
- If a node is down for more than five minutes, its pods are terminated. Pods that belong to a `ReplicaSet` are recreated on other nodes.
- **Pod Eviction Timeout:** how long the controller manager waits for a pod to come back online. The default is five minutes (`tolerationSeconds: 300`).
- In modern Kubernetes, eviction is handled through node taints and pod tolerations. When a node becomes `NotReady` or unreachable, Kubernetes applies `NoExecute` taints. Pods are evicted after a grace period unless they tolerate those taints.

> The node goes offline, and the control plane waits up to five minutes before it considers the node dead.

## Safer Way: Drain, Cordon and Uncordon

### Drain

- Draining moves the workloads off a node so they run on other nodes in the cluster.

```bash
kubectl drain node-name
```

- Pods are gracefully terminated on the drained node and recreated on another node.
- Draining also cordons the node, so no new pods are scheduled on it.
- `DaemonSet` pods cause errors when running `kubectl drain`, so use `--ignore-daemonsets`.
- `--force` is needed when there are standalone pods that no controller manages (such as a Deployment or ReplicaSet). It acknowledges that these pods will be permanently deleted. They are not recreated on other nodes, because no controller manages them.

### Cordon

- Cordoning marks a node as `unschedulable`. No new pods are scheduled on it until the restriction is removed.

```bash
kubectl cordon node-name
```

- Unlike drain, cordon does not terminate or move pods that are already running on the node. It only prevents new pods from being scheduled there.

### Uncordon

- When the work is done, remove the restriction so the node accepts pods again:

```bash
kubectl uncordon node-name
```
