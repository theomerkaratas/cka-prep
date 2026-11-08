# Worker Plane Failure: Unknown Node

When a worker node stops communicating with the control plane, for example after a crash, its conditions are set to `Unknown`. This can mean the node is lost.

## Find When It Stopped

Check the last heartbeat time in the `describe` output. It shows when the node might have crashed.

```bash
kubectl describe node <node> | grep -A8 "Conditions:"
kubectl get node <node> -o jsonpath='{.status.conditions[?(@.type=="Ready")].lastHeartbeatTime}'
```

## Check the Node Itself

1. Is the node online at all, or has it crashed? Log in to the node, or check it from your hypervisor or cloud console.
2. If it crashed, bring it back up.
3. Once it is up, check its resources and kubelet (see [Resources](resources.md) and [kubelet](kubelet.md)).

```bash
ping -c 3 <node-ip>
ssh <node>
```
