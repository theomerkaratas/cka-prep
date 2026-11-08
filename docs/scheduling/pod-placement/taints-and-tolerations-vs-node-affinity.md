# Taints & Tolerations vs. Node Affinity

A common CKA scenario involves dedicating specific nodes to specific workloads (e.g. running GPU workloads only on GPU nodes). Neither Taints & Tolerations nor Node Affinity alone solves this completely.

---

## The Limitation of Each Mechanism

### 1. Taints & Tolerations Alone
- **What it does:** Tainting a node ensures that **other pods cannot be scheduled** onto it unless they have a matching toleration.
- **The problem:** A pod with a matching toleration is **not forced** to go to the tainted node. The scheduler might still place it on any other untainted node in the cluster.

### 2. Node Affinity Alone
- **What it does:** Ensures that designated pods are **only scheduled onto nodes with matching labels**.
- **The problem:** Other pods without node affinity can **still be scheduled** onto these nodes, consuming their resources.

---

## The Solution: Combining Both

To achieve **complete node dedication** (where Pod A runs *only* on Node A, and *only* Pod A can run on Node A), combine both mechanisms:

| Step | Action on Node | Action on Pod | Result |
| :--- | :--- | :--- | :--- |
| **1. Taint & Toleration** | Apply Taint: `color=red:NoSchedule` | Add Toleration: `color=red:NoSchedule` | Prevents non-red pods from running on the red node. |
| **2. Label & Node Affinity** | Apply Label: `color=red` | Add Node Affinity: `color In (red)` | Prevents red pods from running on any non-red node. |

---

## Complete Pod Specification Example

```yaml
# dedicated-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dedicated-red-pod
spec:
  containers:
  - name: app
    image: nginx
  # 1. Toleration: Allows pod onto tainted red node
  tolerations:
  - key: "color"
    operator: "Equal"
    value: "red"
    effect: "NoSchedule"
  # 2. Node Affinity: Forces pod to strictly choose red node
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: color
            operator: In
            values:
            - red
```


