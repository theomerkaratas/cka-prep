# Patches: Dictionaries (Labels)

- **Official Documentation:** [Kustomize Patches Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)

A **dictionary** is a set of key/value pairs, such as `labels`. You can **update**, **add**, or **remove** a key in one. Each of these works with both patch types.

| Goal | JSON 6902 `op` | Strategic merge |
| :--- | :--- | :--- |
| Change a value | `replace` | Write the new value |
| Add a new key | `add` | Write the new key |
| Remove a key | `remove` | Set the key to `null` |

---

## Example Setup

The labels we will change are in the **Pod template** (`spec.template.metadata.labels`). That is why the paths go through `/spec/template/metadata/labels`.

```text
api/
├── deployment.yaml
└── kustomization.yaml
```

```yaml
# api/deployment.yaml
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
        component: api
    spec:
      containers:
      - name: api
        image: nginx
```

---

## 1. Update a Key (JSON 6902)

Goal: change `component: api` to `component: web`.

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- target:
    kind: Deployment
    name: api-deployment
  patch: |-
    - op: replace
      path: /spec/template/metadata/labels/component
      value: web
```

Result: `component: web`

The path goes to the dictionary, then to the key name: `/spec/template/metadata/labels` + `/component`.

Result YAML (`kubectl kustomize api/`):

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
        component: web
    spec:
      containers:
      - image: nginx
        name: api
```

---

## 2. Update a Key (Strategic Merge, Separate File)

Goal: the same change, with the patch in its own file.

```yaml
# api/label-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        component: web
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: label-patch.yaml
```

Result: `component: web`

Keep only what changes, plus `apiVersion`, `kind`, and `metadata.name` so Kustomize knows which object to update.

Result YAML (`kubectl kustomize api/`):

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
        component: web
    spec:
      containers:
      - image: nginx
        name: api
```

---

## 3. Add a Key (JSON 6902)

Goal: add a second label, `org: code-cloud`.

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- target:
    kind: Deployment
    name: api-deployment
  patch: |-
    - op: add
      path: /spec/template/metadata/labels/org
      value: code-cloud
```

Result: the labels are now `app: api`, `component: api`, and `org: code-cloud`.

Result YAML (`kubectl kustomize api/`):

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
        component: api
        org: code-cloud
    spec:
      containers:
      - image: nginx
        name: api
```

---

## 4. Add a Key (Strategic Merge, Separate File)

Goal: add `org: code-cloud` with a separate file.

```yaml
# api/label-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        org: code-cloud
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: label-patch.yaml
```

Result: `org: code-cloud` is merged in with the existing labels.

Result YAML (`kubectl kustomize api/`):

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
        component: api
        org: code-cloud
    spec:
      containers:
      - image: nginx
        name: api
```

---

## 5. Remove a Key (JSON 6902)

Goal: remove the `org` label. This example starts from the labels in step 3 (`app`, `component`, and `org`).

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- target:
    kind: Deployment
    name: api-deployment
  patch: |-
    - op: remove
      path: /spec/template/metadata/labels/org
```

Result: `org` is removed. No `value` is needed.

Result YAML (`kubectl kustomize api/`):

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
        component: api
    spec:
      containers:
      - image: nginx
        name: api
```

---

## 6. Remove a Key (Strategic Merge, Separate File)

Goal: remove the `org` label. Set the key to `null`. This example also starts from the labels in step 3.

```yaml
# api/label-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        org: null
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: label-patch.yaml
```

Result: `org` is removed.

Result YAML (`kubectl kustomize api/`):

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
        component: api
    spec:
      containers:
      - image: nginx
        name: api
```

---

## CKA Quick Reference

- Dictionary paths point to the dictionary, then the key: `.../labels/<key>`.
- JSON 6902: `replace` to change, `add` to add, `remove` to delete.
- Strategic merge: write the key with the new value to change or add it, and `null` to delete it.
- Check your work with `kubectl kustomize <dir>`.
