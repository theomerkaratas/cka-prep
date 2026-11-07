# Manual Scheduling

If a cluster does not have a running scheduler (or if you want to bypass the scheduler), you can manually assign a Pod to a specific node.

---

## 1. Setting `nodeName` in Pod Definition

The `kube-scheduler` assigns a pod to a node by setting the `nodeName` field under `spec`. You can set this field manually at creation time:

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  nodeName: node02 # Manually assigns the pod to node02 directly
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
```

> **Note:** The `nodeName` field can only be set when the pod is created. Once a pod is running, Kubernetes does not allow updating `nodeName` directly.

---

## 2. Scheduling an Existing Pod via Binding Object

If a Pod is already created without a node assigned (in `Pending` state), you cannot simply edit `nodeName`. Instead, you can create a `Binding` object and POST it to the Pod's binding subresource API, which is what the scheduler does internally:

### Binding Definition
```yaml
# binding.yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx # Must match target pod name
target:
  apiVersion: v1
  kind: Node
  name: node02 # Target node
```

### Sending the Binding Request
```bash
# Send binding to Kubernetes API via curl
curl --header "Content-Type:application/json" --request POST \
  --data '{"apiVersion":"v1","kind":"Binding","metadata":{"name":"nginx"},"target":{"apiVersion":"v1","kind":"Node","name":"node02"}}' \
  http://localhost:8001/api/v1/namespaces/default/pods/nginx/binding
```
*(Assumes `kubectl proxy` is running on port 8001, or you authenticate directly with kube-apiserver certificates).*








