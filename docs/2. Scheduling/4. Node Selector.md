# Node Selector

`nodeSelector` is the simplest recommended form of node selection constraint. It specifies a map of key-value pairs that a node must have as labels for the pod to be eligible to run on that node.

---

## How It Works

1. **Label the target node(s):**
   ```bash
   # Syntax: kubectl label nodes <node-name> <key>=<value>
   kubectl label nodes node01 size=Large disktype=ssd
   ```

2. **Add `nodeSelector` in the Pod spec:**
   ```yaml
   # pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: myapp
     labels:
       app: myapp
   spec:
     containers:
     - name: nginx
       image: nginx:alpine
     nodeSelector:
       size: Large
       disktype: ssd
   ```

3. **Verify scheduling:**
   ```bash
   kubectl get pod myapp -o wide
   ```

---

## Removing Labels from Nodes

To remove labels from a node, suffix the label key with a minus (`-`) sign:

```bash
# Remove a single label
kubectl label nodes node01 size-

# Remove multiple labels at once
kubectl label nodes node01 size- disktype-
```

---

## Limitations of `nodeSelector`

While simple, `nodeSelector` has strict limitations:
- Only supports exact string equality matches (`AND` logic across all key-value pairs).
- Cannot express `OR` conditions (e.g. `size == Large OR Medium`).
- Cannot express negative conditions (e.g. `size != Small`).
- Does not support soft preferences (it is strictly hard required).

For these advanced capabilities, use **Node Affinity**.
