# Rolling Updates and Rollbacks

Deployments in Kubernetes manage versioning, rollouts, and rollbacks seamlessly through ReplicaSets.

---

## Deployment Revisions & Rollout Mechanics

- **Rollout:** The process of deploying a new version of the Pod template (e.g. changing container images, environment variables, or resource limits).
- **Deployment Revision:** A recorded snapshot of the Deployment specification. Each rollout creates a new ReplicaSet and assigns a new incremental revision number (1, 2, 3, etc.).
- When an update occurs, the Deployment Controller gradually scales up the new ReplicaSet while scaling down the old ReplicaSet.

---

## Deployment Strategies

1. **`RollingUpdate` (Default):**
   - Gradually replaces old pods with new pods.
   - Ensures **zero downtime**.
   - Controlled by `maxSurge` (how many extra pods can be created over desired count) and `maxUnavailable` (how many pods can be unavailable during update).
2. **`Recreate`:**
   - Immediately terminates all existing pods, and then creates new ones.
   - Causes **temporary downtime**, but guarantees that different application versions never run concurrently.

---

## Updating Deployments

```bash
# Update container image imperatively
kubectl set image deployment/my-deployment nginx=nginx:1.26

# Monitor rollout progression in real time
kubectl rollout status deployment/my-deployment

# View rollout history (list of revisions)
kubectl rollout history deployment/my-deployment

# View details of a specific revision
kubectl rollout history deployment/my-deployment --revision=2
```

---

## Rollbacks (Undo)

If an update fails (e.g., bad image tag, failing readiness probe), you can immediately roll back:

```bash
# Roll back to the immediately preceding revision
kubectl rollout undo deployment/my-deployment

# Roll back to a specific revision from the history
kubectl rollout undo deployment/my-deployment --to-revision=1

# Pause and resume rollout (useful when applying multiple changes in sequence)
kubectl rollout pause deployment/my-deployment
kubectl rollout resume deployment/my-deployment
```

---

## Recording Change Causes

To see meaningful descriptions in `kubectl rollout history`, record the change cause annotation:

```bash
# Annotate deployment with change cause
kubectl annotate deployment/my-deployment kubernetes.io/change-cause="Upgraded nginx to version 1.26" --overwrite
```

