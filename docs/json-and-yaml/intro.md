# Introduction to YAML

YAML is a human-readable data format commonly used for Kubernetes manifests. Kubernetes resources are described with key-value mappings, lists, and scalar values.

## Basic Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

- **Mappings** associate keys with values, such as `kind: Pod`.
- **Sequences** are lists, written with `-` before each item.
- **Scalars** are individual values, such as `nginx` or `80`.
- Indentation defines nesting. Use spaces, not tabs, and keep sibling keys aligned.

## Common Scalar Values

```yaml
replicas: 3
enabled: true
message: "This is a string"
command: ["sh", "-c", "echo hello"]
```

Quote strings when their contents could be interpreted as another type or when they contain special characters. For example, quote version-like values such as `"1.25"` to keep them strings.

## Working with Kubernetes YAML

Generate a starter manifest with `kubectl`, then inspect and edit it:

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

Apply a manifest and verify the resulting resource:

```bash
kubectl apply -f pod.yaml
kubectl describe pod nginx
```