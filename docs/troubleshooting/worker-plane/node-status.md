# Worker Plane Failure: Node Status

Start by checking the node status in the cluster. Are the nodes `Ready` or `NotReady`?

```bash
kubectl get nodes -o wide
kubectl describe node <node>
```

If a node is `NotReady`, read its conditions in the `describe` output. Each condition is `True`, `False`, or `Unknown`.

| Condition | Set to True when |
| :--- | :--- |
| `Ready` | The node is healthy and can run pods |
| `MemoryPressure` | The node is low on memory |
| `DiskPressure` | The disk capacity is low |
| `PIDPressure` | There are too many processes |
| `OutOfDisk` | The node is out of disk space (older clusters) |

To see the conditions only:

```bash
kubectl get node <node> -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
```

If a condition is `Unknown` for the whole node, the node has stopped communicating with the control plane. See [Unknown Node](node-unknown.md).
