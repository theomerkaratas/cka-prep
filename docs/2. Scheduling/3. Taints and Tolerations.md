# Taints and Tolerations

**Taints and Tolerations** work together to ensure that Pods are not scheduled onto inappropriate nodes.

- **Taints** are applied to **Nodes** to repel certain pods.
- **Tolerations** are applied to **Pods** to allow (tolerate) them to be scheduled on tainted nodes.

> **Crucial Rule:** Taints and tolerations **do NOT** force or direct a pod to a specific node. They only allow a pod to tolerate a node that has a taint.

---

## Taint Effects

| Effect | Behavior |
| :--- | :--- |
| **`NoSchedule`** | Pods without matching tolerations will **not** be scheduled on this node. Existing running pods are **unaffected**. |
| **`PreferNoSchedule`** | Soft constraint. The scheduler attempts to avoid placing the pod on this node, but will place it there if no other nodes are available. |
| **`NoExecute`** | Pods without matching tolerations cannot be scheduled, and any **existing pods running on the node without matching tolerations will be evicted immediately** (or after `tolerationSeconds`). |

---

## Managing Node Taints

### Applying a Taint
```bash
# Syntax: kubectl taint nodes <node-name> <key>=<value>:<taint-effect>
kubectl taint nodes node1 tier=frontend:NoSchedule
```

### Inspecting Taints
```bash
kubectl describe node node1 | grep -i taints
```

### Removing a Taint
To remove a taint, specify the taint key and effect followed by a minus (`-`) sign:
```bash
kubectl taint nodes node1 tier=frontend:NoSchedule-
```

---

## Configuring Tolerations on a Pod

### Example 1: `Equal` Operator
Matches exact key, value, and effect:
```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: nginx
    image: nginx:alpine
  tolerations:
  - key: "tier"
    operator: "Equal"
    value: "frontend"
    effect: "NoSchedule"
```

### Example 2: `Exists` Operator
Matches the presence of a key regardless of its value:
```yaml
tolerations:
- key: "tier"
  operator: "Exists"
  effect: "NoSchedule"
```

### Example 3: Eviction Delay with `NoExecute`
```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300 # Wait 5 minutes before evicting
```

