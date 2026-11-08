# Step 1: Check the Pods

A Service can only send traffic to pods that are healthy. So first confirm that the pods are running.

## Commands

```bash
# List the pods for the app using its label
kubectl get pods -l app=hostnames

# Details and events (use a pod name from the list above)
kubectl describe pod <pod-name>

# Application logs
kubectl logs <pod-name>
```

## Read the Output

| Column | What to look for |
| :--- | :--- |
| `READY` | `1/1` means all containers are ready. `0/1` means not ready. |
| `STATUS` | `Running` is what you want. |
| `RESTARTS` | `0` is good. A high number points to a problem inside the pod. |
| `AGE` | How long the pod has been running. |

## What to Do Next

- **Pods are `Running`, `Ready`, and have no restarts:** go to [Step 2](direct-pod-access.md).
- **Pods are not `Running` or have many restarts:** the problem is inside the pod. Use `kubectl describe` for events and `kubectl logs` for the app output. Fix the pod first, then come back to Step 2.
