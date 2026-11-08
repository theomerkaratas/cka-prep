
# Resource Quotas

A **`ResourceQuota`** provides constraints that limit aggregate resource consumption per namespace. It can limit the quantity of objects that can be created in a namespace by type, as well as the total amount of compute resources that may be consumed by resources in that namespace.

---

## LimitRange vs. ResourceQuota

- **`LimitRange`**: Constrains individual Pods and Containers (per-pod / per-container limit).
- **`ResourceQuota`**: Constrains the aggregate total across all workloads in the entire namespace.

> **Crucial Rule:** If a `ResourceQuota` defines limits for `requests.cpu` or `requests.memory`, **every pod in that namespace MUST declare requests/limits**. If a pod omits them, its creation is rejected—unless a `LimitRange` exists in that namespace to inject defaults automatically.

---

## Declarative ResourceQuota Manifest

```yaml
# resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-and-count-quota
  namespace: dev
spec:
  hard:
    # Compute resources
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "10"
    limits.memory: 10Gi
    # Storage resources
    requests.storage: 50Gi
    persistentvolumeclaims: "5"
    # Object count limits
    pods: "10"
    services: "5"
    configmaps: "10"
    secrets: "10"
```

---

## Imperative Commands

```bash
# Create a ResourceQuota imperatively
kubectl create quota dev-quota \
  --hard=cpu=4,memory=4Gi,pods=10 \
  --namespace=dev

# Generate YAML template
kubectl create quota dev-quota \
  --hard=cpu=4,memory=4Gi,pods=10 \
  --dry-run=client -o yaml > quota.yaml

# Check quota usage and limits (Used vs. Hard)
kubectl describe resourcequota dev-quota -n dev
```

Sample output:
```text
Name:                    dev-quota
Namespace:               dev
Resource                 Used  Hard
--------                 ----  ----
configmaps               1     10
limits.cpu               0     10
limits.memory            0     10Gi
pods                     2     10
requests.cpu             500m  4
requests.memory          512Mi 4Gi
```

