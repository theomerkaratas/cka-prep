# LimitRange

A **`LimitRange`** is a policy to constrain resource allocations (for Pods or Containers) in a specific namespace.

It allows administrators to:
- Enforce minimum and maximum compute resource requirements (CPU / Memory).
- Automatically set default requests and limits for containers that do not explicitly declare them.
- Enforce a limit-to-request ratio.

---

## Combined CPU and Memory LimitRange Manifest

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-cpu-limitrange
  namespace: dev
spec:
  limits:
  - type: Container
    default:           # Default limit if container omits it
      cpu: 500m
      memory: 512Mi
    defaultRequest:    # Default request if container omits it
      cpu: 250m
      memory: 256Mi
    max:               # Maximum allowed limit
      cpu: "1"
      memory: 1Gi
    min:               # Minimum required request
      cpu: 100m
      memory: 128Mi
```

---

## Key Rules to Remember

1. **Namespace Scope:** `LimitRange` is enforced strictly at the namespace level.
2. **Existing Pods:** Creating or updating a `LimitRange` **does not affect already running pods**; it only applies to newly created pods.
3. **Validation:** If a user tries to create a pod exceeding `max` or below `min`, the `kube-apiserver` rejects the creation with an admission error.
4. **Injection:** If a pod is created without `resources.requests` or `resources.limits`, the admission controller automatically injects the `defaultRequest` and `default` values defined in the `LimitRange`.

---

## Verifying LimitRange

```bash
# Describe the LimitRange in a namespace
kubectl describe limitrange mem-cpu-limitrange -n dev
```

