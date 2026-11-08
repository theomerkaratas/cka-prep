# Components

- **Official Documentation:** [Kustomize Components](https://kubectl.docs.kubernetes.io/guides/config_management/components/)

A **component** is a reusable set of config for one optional feature (for example, a database or a cache). You write it once, then turn it on in only the overlays that need it.

| Where | What goes in it | Example |
| :--- | :--- | :--- |
| **base** | Config every overlay needs | The API Deployment |
| **components** | Config for one optional feature | Postgres, Redis |
| **overlays** | Pick which components to turn on | `dev`, `premium`, `standalone` |

Without components, you would copy the same YAML into every overlay that needs the feature. Copies drift apart over time. A component is written once and included by reference.

---

## Component vs Base vs Resources

- A **base** is always included.
- A **component** is included only when an overlay lists it under `components:`.
- A **component** must declare `kind: Component`. A normal `kustomization.yaml` (a base or an overlay) uses `kind: Kustomization`, which is optional.

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
```

---

## Folder Structure

```text
k8s/
├── base/
│   ├── kustomization.yaml
│   └── api-depl.yaml
├── components/
│   ├── caching/
│   │   ├── kustomization.yaml
│   │   ├── redis-depl.yaml
│   │   └── deployment-patch.yaml
│   └── db/
│       ├── kustomization.yaml
│       ├── postgres-depl.yaml
│       └── deployment-patch.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── premium/
    │   └── kustomization.yaml
    └── standalone/
        └── kustomization.yaml
```

Which overlay uses what:

| Overlay | Base | `db` component | `caching` component |
| :--- | :---: | :---: | :---: |
| **dev** | yes | yes | no |
| **premium** | yes | yes | yes |
| **standalone** | yes | no | yes |

---

## Base

Shared config. Included in every overlay.

```yaml
# k8s/base/kustomization.yaml
resources:
- api-depl.yaml
```

```yaml
# k8s/base/api-depl.yaml
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
      - name: api
        image: nginx
```

---

## Component: `db`

The `db` component adds a Postgres Deployment, a Secret, and a patch that gives the API the database password.

```yaml
# k8s/components/db/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- postgres-depl.yaml

secretGenerator:
- name: postgres-cred
  literals:
  - password=postgres123

patches:
- path: deployment-patch.yaml
```

```yaml
# k8s/components/db/postgres-depl.yaml
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
      - name: postgres
        image: postgres
```

```yaml
# k8s/components/db/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
      - name: api
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-cred
              key: password
```

- `secretGenerator` creates the Secret `postgres-cred`.
- The patch adds the `DB_PASSWORD` env var to the API container. It reads from that Secret.

---

## Component: `caching`

The `caching` component adds a Redis Deployment and tells the API where to find it.

```yaml
# k8s/components/caching/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- redis-depl.yaml

patches:
- path: deployment-patch.yaml
```

```yaml
# k8s/components/caching/redis-depl.yaml
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
      - name: redis
        image: redis
```

```yaml
# k8s/components/caching/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
      - name: api
        env:
        - name: CACHE_HOST
          value: redis-deployment
```

---

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
