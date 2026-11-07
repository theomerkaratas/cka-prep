# Namespaces

**Namespaces** provide a mechanism for isolating groups of resources within a single Kubernetes cluster. Resource names must be unique within a namespace, but not across namespaces.

---

## DNS and Cross-Namespace Communication

Within the cluster, Kubernetes CoreDNS creates DNS records for Services:

- **Same Namespace:** A Pod can reach a service by its simple name: `backend-service`
- **Different Namespace:** A Pod must include the target namespace: `backend-service.<namespace>`
- **Fully Qualified Domain Name (FQDN):**
  ```text
  <service-name>.<namespace-name>.svc.cluster.local
  ```
  *(Example: `db-service.prod.svc.cluster.local`)*

---

## Working with Namespaces

### Listing Resources
```bash
# List pods in a specific namespace
kubectl get pods -n kube-system

# List pods across ALL namespaces
kubectl get pods --all-namespaces
# or shorthand:
kubectl get pods -A
```

### Creating Resources in a Namespace
```bash
# Create a namespace imperatively
kubectl create namespace dev

# Run a pod inside a namespace
kubectl run my-app --image=nginx -n dev

# Apply a manifest into a specific namespace (overrides manifest namespace)
kubectl apply -f pod.yaml -n dev
```

Declarative Pod with namespace:
```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  namespace: dev
  labels:
    app: myapp
spec:
  containers:
  - name: nginx
    image: nginx
```

---

## Changing the Default Namespace in Context

Instead of appending `-n <namespace>` to every command, you can permanently switch your current context's default namespace:

```bash
# Switch default namespace to 'dev'
kubectl config set-context --current --namespace=dev

# Verify current context settings
kubectl config view --minify | grep namespace
```

---

## Resource Quotas per Namespace

To prevent any single team or environment from exhausting cluster resources, apply a `ResourceQuota` to the namespace:

```yaml
# compute-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```

When a ResourceQuota is applied to a namespace, pods created in that namespace must specify resource requests and limits (or have them injected automatically via a `LimitRange`).

