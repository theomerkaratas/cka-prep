# Deployments

A **Deployment** provides declarative updates for Pods and ReplicaSets. You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate.

---

## Deployment Strategies

When updating pod templates (e.g. updating a container image), Deployments support two primary strategies:

1. **`RollingUpdate` (Default):**
   - Updates pods progressively with zero downtime.
   - Controlled by `maxSurge` (how many extra pods can be created above the desired replica count) and `maxUnavailable` (how many pods can be unavailable during the update).
2. **`Recreate`:**
   - Terminates all running pods before creating new ones.
   - Causes application downtime, but ensures that old and new versions never run simultaneously (useful for legacy databases or apps that don't support parallel schema migrations).

---

## Declarative Deployment Manifest

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: myapp
    tier: frontend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  selector:
    matchLabels:
      app: myapp
      tier: frontend
  template:
    metadata:
      labels:
        app: myapp
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

---

## Imperative Commands & Dry-Run Generation

In the CKA exam, generating YAML definitions quickly with `--dry-run=client -o yaml` is essential:

```bash
# Create a deployment imperatively
kubectl create deployment my-deployment --image=nginx:1.25 --replicas=3

# Generate YAML template without creating the resource
kubectl create deployment my-deployment --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > deployment.yaml

# Generate Pod YAML template
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

---

## Managing Rollouts and Revisions

```bash
# Update container image
kubectl set image deployment/my-deployment nginx=nginx:1.26

# Monitor rollout status
kubectl rollout status deployment/my-deployment

# View rollout history (revision history)
kubectl rollout history deployment/my-deployment

# View details of a specific revision
kubectl rollout history deployment/my-deployment --revision=2

# Undo the rollout (revert to previous revision)
kubectl rollout undo deployment/my-deployment

# Rollback to a specific revision
kubectl rollout undo deployment/my-deployment --to-revision=1

# Pause and resume rollout (useful for making multiple changes)
kubectl rollout pause deployment/my-deployment
kubectl rollout resume deployment/my-deployment
```
