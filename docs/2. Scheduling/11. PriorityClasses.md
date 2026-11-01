# Priority Classes

A **`PriorityClass`** is a non-namespaced (cluster-scoped) object that defines a mapping from a priority class name to an integer value. Larger numbers indicate higher priority.

When a pod cannot be scheduled due to insufficient node resources, the scheduler can preempt (evict) lower-priority pods to schedule the higher-priority pod.

---

## Defining a PriorityClass

```yaml
# priorityclass.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false # If true, pods without priorityClassName receive this priority (only one global default allowed)
preemptionPolicy: PreemptLowerPriority # Options: PreemptLowerPriority (default) or Never
description: "Mission-critical production workloads"
```

---

## Preemption Policies

| Policy | Behavior |
| :--- | :--- |
| **`PreemptLowerPriority`** *(Default)* | The scheduler will evict lower-priority running pods to free up node resources for this high-priority pod. |
| **`Never`** | **Non-preempting.** The scheduler will **not** evict running pods. The high-priority pod will wait in the scheduling queue until resources become free naturally, but will be scheduled ahead of lower-priority pending pods. |

---

## Using PriorityClass in a Pod

Attach the priority class to the Pod specification via `spec.priorityClassName`:

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-critical
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "1"
        memory: 1Gi
```

---

## Built-in System Priority Classes

Kubernetes ships with two built-in, reserved priority classes for critical control plane and system pods:

1. **`system-cluster-critical`** (value: `2000000000`) - Used by components like CoreDNS.
2. **`system-node-critical`** (value: `2000001000`) - Used by components like kube-proxy and storage plugins that must never be evicted from a node.

---

## Inspecting Priority Classes

```bash
# List all PriorityClasses
kubectl get priorityclasses
# Shorthand:
kubectl get pc

# View pods alongside their assigned priority class
kubectl get pods -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName"
```
