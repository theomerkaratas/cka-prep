# Overlays

- **Official Documentation:** [Kustomize Bases and Overlays](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#overlay)

An **overlay** lets you use one set of shared YAML for every environment, and change only what differs per environment.

- **base/**: the shared config (the defaults).
- **overlays/dev/**, **overlays/prod/**: one folder per environment. Each one imports the base and adds patches.

The base is never edited. Each environment gets its own result.

---

## Folder Structure

```text
k8s/
├── base/
│   ├── kustomization.yaml
│   └── nginx-deployment.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── grafana-deployment.yaml
```

---

## Base

The base holds the shared config. Here the Deployment has `replicas: 1`.

```yaml
# k8s/base/nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

```yaml
# k8s/base/kustomization.yaml
resources:
- nginx-deployment.yaml
```

---

## Dev Overlay

Dev uses `replicas: 2`.

The `resources` line points up two folders to `base/`. `../` means "go up one folder."

```yaml
# k8s/overlays/dev/kustomization.yaml
resources:
- ../../base

patches:
- target:
    kind: Deployment
    name: nginx-deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 2
```

Result YAML (`kubectl kustomize k8s/overlays/dev`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

---

## Prod Overlay

Prod uses `replicas: 3`, and it also adds a new Grafana Deployment that exists only in prod.

```yaml
# k8s/overlays/prod/kustomization.yaml
resources:
- ../../base
- grafana-deployment.yaml

patches:
- target:
    kind: Deployment
    name: nginx-deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 3
```

```yaml
# k8s/overlays/prod/grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana
```

Result YAML (`kubectl kustomize k8s/overlays/prod`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - image: grafana/grafana
        name: grafana
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

The `---` line separates two objects in one output. Grafana appears only here because only prod adds it.

---

## Complex Scenario: Multiple Apps in Subfolders

Real projects have several apps (here `db` and `api`), each with its own folder. Each overlay has a folder per app, and each app folder has its own patch.

```text
k8s/
├── base/
│   ├── kustomization.yaml          # resources: db, api
│   ├── db/
│   │   ├── db-depl.yaml
│   │   ├── db-service.yaml
│   │   └── kustomization.yaml
│   └── api/
│       ├── api-depl.yaml
│       ├── api-service.yaml
│       └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml      # resources: db, api
    │   ├── db/
    │   │   ├── db-patch.yaml
    │   │   └── kustomization.yaml
    │   └── api/
    │       ├── api-patch.yaml
    │       └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml      # resources: db, api + env label
        ├── db/
        │   ├── db-patch.yaml
        │   └── kustomization.yaml
        └── api/
            ├── api-patch.yaml
            └── kustomization.yaml
```

The pattern has three levels. Each level only lists the level directly below it:

1. **`base/kustomization.yaml`** lists the app folders (`db`, `api`).
2. **`base/db/kustomization.yaml`** lists that app's files.
3. **`overlays/<env>/db/kustomization.yaml`** imports the base app folder, then applies that env's patch.

### Base

```yaml
# k8s/base/kustomization.yaml
resources:
- db
- api
```

```yaml
# k8s/base/db/kustomization.yaml
resources:
- db-depl.yaml
- db-service.yaml
```

```yaml
# k8s/base/db/db-depl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: postgres:13
        ports:
        - containerPort: 5432
```

```yaml
# k8s/base/db/db-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  selector:
    app: db
  ports:
  - port: 5432
    targetPort: 5432
```

```yaml
# k8s/base/api/kustomization.yaml
resources:
- api-depl.yaml
- api-service.yaml
```

```yaml
# k8s/base/api/api-depl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-depl
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
        ports:
        - containerPort: 80
```

```yaml
# k8s/base/api/api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80
```

### Dev Overlay

Dev changes the db image to `postgres:15` and runs 2 API replicas.

```yaml
# k8s/overlays/dev/kustomization.yaml
resources:
- db
- api
```

```yaml
# k8s/overlays/dev/db/kustomization.yaml
resources:
- ../../../base/db

patches:
- path: db-patch.yaml
```

```yaml
# k8s/overlays/dev/db/db-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-depl
spec:
  template:
    spec:
      containers:
      - name: db
        image: postgres:15
```

```yaml
# k8s/overlays/dev/api/kustomization.yaml
resources:
- ../../../base/api

patches:
- path: api-patch.yaml
```

```yaml
# k8s/overlays/dev/api/api-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-depl
spec:
  replicas: 2
```

Result YAML (`kubectl kustomize k8s/overlays/dev`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: api
---
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: db
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-depl
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - image: nginx
        name: api
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - image: postgres:15
        name: db
        ports:
        - containerPort: 5432
```

### Prod Overlay

Prod runs 2 DB replicas with `postgres:15`, 5 API replicas, and adds the label `env: prod` to every resource. The label is added in the root file, so it applies to both apps.

```yaml
# k8s/overlays/prod/kustomization.yaml
resources:
- db
- api

labels:
- pairs:
    env: prod
  includeSelectors: false
```

```yaml
# k8s/overlays/prod/db/kustomization.yaml
resources:
- ../../../base/db

patches:
- path: db-patch.yaml
```

```yaml
# k8s/overlays/prod/db/db-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-depl
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: db
        image: postgres:15
```

```yaml
# k8s/overlays/prod/api/kustomization.yaml
resources:
- ../../../base/api

patches:
- path: api-patch.yaml
```

```yaml
# k8s/overlays/prod/api/api-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-depl
spec:
  replicas: 5
```

Result YAML (`kubectl kustomize k8s/overlays/prod`):

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    env: prod
  name: api-service
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: api
---
apiVersion: v1
kind: Service
metadata:
  labels:
    env: prod
  name: db-service
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: db
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    env: prod
  name: api-depl
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - image: nginx
        name: api
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    env: prod
  name: db-depl
spec:
  replicas: 2
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - image: postgres:15
        name: db
        ports:
        - containerPort: 5432
```

Notice:

- `kubectl kustomize` sorts the output by kind and name, so the Services appear first.
- The `labels` transformer in the prod root file adds `env: prod` to every object it finds below it.

---

## Deploy an Overlay

Point `kubectl` at the overlay folder, not at the base:

```bash
kubectl apply -k k8s/overlays/dev
```

```bash
kubectl apply -k k8s/overlays/prod
```

---

## Notes

- **`resources` vs `bases`:** older guides use `bases:` to import the base. Use `resources:` instead, as above.
- **Overlay folders can add new resources.** List them under `resources:` next to the base, as the Grafana file does.
- **Subfolders are fine.** Base and overlays can each be split into subfolders. Each folder that is listed needs its own `kustomization.yaml`.

---

## CKA Quick Reference

- `base/` = shared config. `overlays/<env>/` = per-environment patches and extras.
- In an overlay, import the base with `- ../../base` (from `overlays/<env>/`).
- Patches in an overlay change the base copy only for that environment.
- Apply an overlay with `kubectl apply -k <overlay-dir>`.
- Check your work with `kubectl kustomize <overlay-dir>`.
