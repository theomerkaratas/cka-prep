# Components: Definitions

- **Official Documentation:** [Kustomize Components](https://kubectl.docs.kubernetes.io/guides/config_management/components/)

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

## CKA Quick Reference

- Each component is a folder with its own `kustomization.yaml` and `kind: Component`.
- Back to: [Intro](intro.md). Next: [Overlays](overlays.md).
