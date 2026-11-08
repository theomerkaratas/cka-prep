# Step 3: Check the Service and Endpoints

The Service must select the healthy pods, and its ports must match the pod ports.

## Step 3.1: Check the Service Definition

```bash
kubectl get svc hostnames -o yaml
```

Check these three fields:

| Field | Must match |
| :--- | :--- |
| `spec.selector` | The labels on the pods (for example, `app: hostnames`) |
| `spec.ports[].port` | The port clients use |
| `spec.ports[].targetPort` | The container port on the pod |

## Step 3.2: Check the Endpoints

Endpoints are the pod IPs and ports that the Service sends traffic to.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=hostnames
kubectl get endpoints hostnames
```

## Step 3.3: Read the Result

- **Endpoints show pod IPs:** the Service finds the pods. Go to [Step 4](coredns.md).
- **Endpoints show `<none>`:** the selector does not match any running pod. Compare `spec.selector` with the pod labels:

```bash
kubectl get pods --show-labels
```

Fix the label or the selector, then check the endpoints again.
