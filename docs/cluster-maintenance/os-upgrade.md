- Scenarios of the take down nodes as part of our cluster (purposes like upgrading base software or applying patches)
- If the node was down for more than five minutes, then the pods are terminated from that node. If the pods were part of a `ReplicaSet`, then they are recreated on other nodes
- **Pod Eviction Timeout:** The time waits for a pod to come back online. It can be set on controller manager with a default value of five minutes (`tolerationSeconds: 300`).
- In modern k8s, pod eviction is handled through node taints and pod tolerations. When a node becomes `NotReady` or unreachable, k8s applies `NoExecute` taints and pods are evicted after a grace period unless they tolerate those taints.

> So the node goes offline, the master node waits for up to five minutes before considering the node dead.

### Safer Way
- We can drain the node of all the workloads, so that the workloads are moved to other nodes in the cluster.
```sh
kubectl drain node-name
```
- when we drain a node, the pods are gracefully terminated from the node that they're on and recreated on another node
- if there are `DaemonSet` pods in the pod, there will be errors when we execute `kubectl drain`. so we should use `--ignore-daemonsets` flag ``
- the node is also cordoned or marked as `unschedulable`. meaning no pods can be scheduled on this node until you specifically remove the restriction. so when we are done, we should execute the command below:
```sh
kubectl uncordon node-name
```
cordon simply marks a node `unschedulable`:
```sh
kubectl cordon node-name
```
- unlike drain, cordon does not terminate or move the pods on an existing node. it simply makes sure that new pods are not scheduled on that node.
- `--force` is used when there are standalone pods on the node not managed by a controller (like Deployment, ReplicaSet etc.), forcing the `drain` by acknowledging that these pods will be permanently deleted and not rescheduled. but after that this ones will not recreate in one of the other nodes. because they are not managed by any controller