# Patches: Lists (Containers)

- **Official Documentation:** [Kustomize Patches Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)

A **list** is an ordered set of items, such as `containers`. You can **update**, **add**, or **remove** an item. The two patch types find items differently.

| Goal | JSON 6902 | Strategic merge |
| :--- | :--- | :--- |
| Change an item | `replace` on the item's index (`/containers/0/...`) | Write the item with its `name` |
| Add an item | `add` with `/containers/-` (end of list) | Write a new item with a new `name` |
| Remove an item | `remove` on the item's index (`/containers/1`) | Write the item with `$patch: delete` |

- In JSON 6902, list items are numbered from `0`.
- In strategic merge, items are matched by their **`name`**.

---

## Example Setup

The Deployment has two containers: `api` (index `0`) and `logger` (index `1`).

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
    spec:
      containers:
      - name: api
        image: nginx
      - name: logger
        image: busybox
```

---

## 1. Change an Item (JSON 6902)

Goal: change the image of the first container (`api`) to `haproxy:2.4`.

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
      path: /spec/template/spec/containers/0/image
      value: haproxy:2.4
```

Result: `api` now uses `haproxy:2.4`.

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
    spec:
      containers:
      - image: haproxy:2.4
        name: api
      - image: busybox
        name: logger
```

---

## 2. Add an Item (JSON 6902)

Goal: add a third container called `sidecar` to the end of the list.

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
      path: /spec/template/spec/containers/-
      value:
        name: sidecar
        image: alpine:3.20
```

Result: `sidecar` is added after `logger`. The `-` means "append."

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
    spec:
      containers:
      - image: nginx
        name: api
      - image: busybox
        name: logger
      - image: alpine:3.20
        name: sidecar
```

---

## 3. Remove an Item (JSON 6902)

Goal: remove the second container (`logger`, index `1`).

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
      path: /spec/template/spec/containers/1
```

Result: `logger` is removed. No `value` is needed.

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
    spec:
      containers:
      - image: nginx
        name: api
```

---

## 4. Change an Item (Strategic Merge, Separate File)

Goal: change the image of `api` to `haproxy:2.4`. Only the `name` and the changed field are needed.

```yaml
# api/container-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
      - name: api
        image: haproxy:2.4
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: container-patch.yaml
```

Result: `api` now uses `haproxy:2.4`. `logger` is untouched.

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
    spec:
      containers:
      - image: haproxy:2.4
        name: api
      - image: busybox
        name: logger
```

---

## 5. Add an Item (Strategic Merge, Separate File)

Goal: add a `sidecar` container. Use a `name` that does not exist yet.

```yaml
# api/new-container-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
      - name: sidecar
        image: alpine:3.20
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: new-container-patch.yaml
```

Result: `sidecar` is added. Strategic merge places new items at the **start** of the list, so `sidecar` comes first.

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
    spec:
      containers:
      - image: alpine:3.20
        name: sidecar
      - image: nginx
        name: api
      - image: busybox
        name: logger
```

---

## 6. Remove an Item (Strategic Merge, Separate File)

Goal: remove the `logger` container. Use `$patch: delete` on the item's `name`.

```yaml
# api/delete-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
      - name: logger
        $patch: delete
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: delete-patch.yaml
```

Result: `logger` is removed.

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
    spec:
      containers:
      - image: nginx
        name: api
```

---

## CKA Quick Reference

- JSON 6902 lists are indexed from `0`. Use `/-` to add to the end.
- Strategic merge matches list items by `name`. Always include it.
- To delete a list item with strategic merge, use `$patch: delete`.
- New strategic merge items go to the start of the list.
- Check your work with `kubectl kustomize <dir>`.
