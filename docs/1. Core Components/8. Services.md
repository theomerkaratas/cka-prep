# Services

A **Service** is an abstract way to expose an application running on a set of Pods as a network service. It provides a stable IP address and DNS name that load-balances traffic across matching Pods.

---

## Service Types

| Type | Description | Use Case |
| :--- | :--- | :--- |
| **`ClusterIP`** | Exposes the Service on an internal cluster IP (default). | Internal communication between services within the cluster. |
| **`NodePort`** | Exposes the Service on each Node's IP at a static port (default range: **`30000-32767`**). | Direct external access without a cloud load balancer. |
| **`LoadBalancer`** | Provisions an external load balancer in supported cloud providers (AWS, GCP, Azure). | Production external access with cloud-managed routing. |
| **`ExternalName`** | Maps the Service to the contents of the `externalName` field (returns a CNAME record). | Accessing external services outside Kubernetes via internal DNS. |

---

## Port Terminology

- **`port`**: The port exposed by the Service internally within the cluster.
- **`targetPort`**: The port on the container inside the backend Pod where traffic is forwarded (defaults to the same value as `port` if omitted).
- **`nodePort`**: The static port allocated on every node in the cluster (valid range: `30000–32767`).

---

## 1. ClusterIP (Default)

Used for communication between microservices within the cluster:

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  ports:
  - port: 80         # Port on the Service
    targetPort: 8080 # Port on the Pod container
  selector:
    app: myapp
    tier: backend
```

---

## 2. NodePort

Exposes the service externally on `<NodeIP>:<NodePort>`:

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  ports:
  - port: 80         # Service port
    targetPort: 80   # Pod container port
    nodePort: 30008  # Static port on each node (30000-32767)
  selector:
    app: myapp
    tier: frontend
```

---

## 3. LoadBalancer

Automatically creates a cloud load balancer routing to the `NodePort` and `ClusterIP`:

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: public-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30008  # Optional: Kubernetes auto-allocates if omitted
  selector:
    app: myapp
    tier: frontend
```

---

## Essential Imperative Commands

```bash
# Expose a deployment as a ClusterIP service
kubectl expose deployment backend --port=80 --target-port=8080 --name=backend-svc

# Expose a deployment as a NodePort service
kubectl expose deployment frontend --type=NodePort --port=80 --target-port=80 --name=frontend-svc

# Generate service manifest using dry-run
kubectl expose deployment my-deployment --type=NodePort --port=80 --dry-run=client -o yaml > svc.yaml

# List services and their Endpoints
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
```


