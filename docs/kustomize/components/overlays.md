# Components: Overlays

- **Official Documentation:** [Kustomize Components](https://kubectl.docs.kubernetes.io/guides/config_management/components/)

## Overlays

An overlay imports the base with `resources:` and turns on components with `components:`. Paths are relative to the overlay's `kustomization.yaml`.

### Dev: Base + `db`

```yaml
# k8s/overlays/dev/kustomization.yaml
resources:
- ../../base

components:
- ../../components/db
```

Result YAML (`kubectl kustomize k8s/overlays/dev`):

```yaml
apiVersion: v1
data:
  password: cG9zdGdyZXMxMjM=
kind: Secret
metadata:
  name: postgres-cred-2k8b68cdbb
type: Opaque
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: password
              name: postgres-cred-2k8b68cdbb
        image: nginx
        name: api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: postgres
  template:
    metadata:
      labels:
        component: postgres
    spec:
      containers:
      - image: postgres
        name: postgres
```

### Premium: Base + `db` + `caching`

```yaml
# k8s/overlays/premium/kustomization.yaml
resources:
- ../../base

components:
- ../../components/db
- ../../components/caching
```

Result YAML (`kubectl kustomize k8s/overlays/premium`):

```yaml
apiVersion: v1
data:
  password: cG9zdGdyZXMxMjM=
kind: Secret
metadata:
  name: postgres-cred-2k8b68cdbb
type: Opaque
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - env:
        - name: CACHE_HOST
          value: redis-deployment
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: password
              name: postgres-cred-2k8b68cdbb
        image: nginx
        name: api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: postgres
  template:
    metadata:
      labels:
        component: postgres
    spec:
      containers:
      - image: postgres
        name: postgres
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: redis
  template:
    metadata:
      labels:
        component: redis
    spec:
      containers:
      - image: redis
        name: redis
```

### Standalone: Base + `caching`

```yaml
# k8s/overlays/standalone/kustomization.yaml
resources:
- ../../base

components:
- ../../components/caching
```

Result YAML (`kubectl kustomize k8s/overlays/standalone`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - env:
        - name: CACHE_HOST
          value: redis-deployment
        image: nginx
        name: api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: redis
  template:
    metadata:
      labels:
        component: redis
    spec:
      containers:
      - image: redis
        name: redis
```

---

## Notes

- **Hash suffix on the Secret.** `kubectl kustomize` adds a hash to generated Secret names (`postgres-cred-2k8b68cdbb`). It also updates every reference to the name, including `secretKeyRef`. You don't write the hash yourself.
- **Plain-text passwords.** The `password=postgres123` line is for learning only. In real projects, don't commit passwords to Git.
- **`components` vs `resources`.** Use `resources:` for things that are always included, and `components:` for optional features.

---

## CKA Quick Reference

- A component has `apiVersion: kustomize.config.k8s.io/v1alpha1` and `kind: Component`.
- Turn a component on in an overlay with `components:`.
- Keep the base for things every overlay needs.
- Check your work with `kubectl kustomize <overlay-dir>`.
