# Node Affinity and Anti-Affinity

**Node Affinity** conceptually resembles `nodeSelector`, but allows you to specify much more expressive selection rules using operators (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`) and soft preferences.

---

## Types of Affinity Rules

Kubernetes defines affinity behavior across two lifecycle stages: **Scheduling** and **Execution**:

| Affinity Type | Meaning |
| :--- | :--- |
| **`requiredDuringSchedulingIgnoredDuringExecution`** | **Hard requirement.** If no matching node is found, the pod remains in `Pending` state. If node labels change after scheduling, the pod continues running. |
| **`preferredDuringSchedulingIgnoredDuringExecution`** | **Soft preference.** The scheduler attempts to place the pod on a matching node, but places it elsewhere if none are available. Uses a `weight` (1–100). |

---

## 1. Node Affinity Examples

### Hard Requirement (`requiredDuringScheduling...`)
```yaml
# pod-node-affinity-required.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In # Options: In, NotIn, Exists, DoesNotExist, Gt, Lt
            values:
            - Large
            - Medium
```

### Soft Preference with Weight (`preferredDuringScheduling...`)
```yaml
# pod-node-affinity-preferred.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: size
            operator: In
            values:
            - Large
```

---

## 2. Pod Affinity & Pod Anti-Affinity

While Node Affinity targets **node labels**, Pod Affinity and Pod Anti-Affinity target **pod labels** to co-locate or spread workloads across a topology domain (`topologyKey`).

### Pod Anti-Affinity (Spreading Pods across Nodes)
Ensures that two pods with the label `app=myapp` are never placed on the same node (ideal for high availability):

```yaml
# pod-anti-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  containers:
  - name: web
    image: nginx
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web-app
        topologyKey: "kubernetes.io/hostname"
```





