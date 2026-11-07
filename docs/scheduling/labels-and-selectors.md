# Labels and Selectors

Labels and Selectors are the standard mechanism used in Kubernetes to group, organize, and select subsets of objects.

---

## Labels vs. Annotations

| Feature | Labels | Annotations |
| :--- | :--- | :--- |
| **Purpose** | Identifying and grouping resources. | Storing non-identifying descriptive metadata. |
| **Selectable?** | **Yes** — queried via selectors (`--selector` / `-l`). | **No** — cannot be used to filter resources via selectors. |
| **Use Cases** | `env=prod`, `app=frontend`, `tier=db` | Build timestamps, git commit SHAs, release notes, ingress configs. |

---

## Filtering Resources with Selectors

You can use the `--selector` or shorthand `-l` flag with `kubectl`:

```bash
# Basic single label match
kubectl get pods -l env=prod

# Multiple labels (Comma-separated acts as a logical AND)
kubectl get pods -A --selector tier=frontend,env=prod,bu=finance
```

### Equality-based vs. Set-based Selectors

```bash
# Equality-based (=, ==, !=)
kubectl get pods -l 'env=prod,tier!=frontend'

# Set-based (in, notin, exists)
kubectl get pods -l 'env in (prod, staging)'
kubectl get pods -l 'tier notin (frontend, cache)'
kubectl get pods -l 'app' # Pods that have the 'app' label (any value)
```

Sample output:
```text
$ kubectl get pods -A --selector tier=frontend,env=prod,bu=finance
NAMESPACE   NAME          READY   STATUS    RESTARTS   AGE
default     app-1-zzxdf   1/1     Running   0          6m34s
```

---

## Managing Labels on Resources

```bash
# Add a label to a pod
kubectl label pod nginx-pod env=prod

# Overwrite an existing label
kubectl label pod nginx-pod env=dev --overwrite

# Remove a label (suffix with a minus sign)
kubectl label pod nginx-pod env-
```
