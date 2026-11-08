# Patches: Add

- **Official Documentation:** [Kustomize Patches Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)

**Add** puts something new into the object: a new key in a dictionary (like `labels`) or a new item in a list (like `containers`).

| Goal | JSON 6902 | Strategic merge |
| :--- | :--- | :--- |
| Add a key | `op: add` on `.../labels/<key>` | Write the new key |
| Add a list item | `op: add` on `.../containers/-` | Write a new item with a new `name` |

- JSON 6902 `add` needs a `value`.
- Strategic merge matches list items by **`name`**. A new name means a new item.

For the setup, see [Patches Intro](intro.md#example-setup).

---

## Add a Key

### 1. JSON 6902

Goal: add the label `org: code-cloud` to the Pod template.

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

### 2. Strategic Merge

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
        org: code-cloud
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: label-patch.yaml
```

Result: `org: code-cloud` is merged in with the existing labels. You do not need to repeat `app` or `component`.

---

## Add a List Item

### 1. JSON 6902

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

### 2. Strategic Merge

Goal: add a `sidecar` container. Use a `name` that does not exist yet.

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
      - name: sidecar
        image: alpine:3.20
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: container-patch.yaml
```

Result: `sidecar` is added, but it goes to the **start** of the list, not the end.

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
      - image: alpine:3.20
        name: sidecar
      - image: nginx
        name: api
      - image: busybox
        name: logger
```

**Gotcha:** if you need `sidecar` last, use JSON 6902 with `/-`.

---

## Add an Environment Variable

In the example Deployment, the first container is `api` (index `0`). If it does not have an `env` field yet, add the field with a list value:

```yaml
patches:
- target:
    kind: Deployment
    name: api-deployment
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/env
      value:
        - name: LOG_LEVEL
          value: info
```

If `env` already exists as a list, append the variable to it instead:

```yaml
- op: add
  path: /spec/template/spec/containers/0/env/-
  value:
    name: LOG_LEVEL
    value: info
```

The `-` appends to the end of the list. Replace `0` with the target container's list index when patching a different container.

---

## CKA Quick Reference

- Dictionary `add`: the path ends with the new key name.
- List `add` (JSON 6902): `/containers/-` appends to the end.
- Add an environment variable under `/containers/<index>/env`; use `/env/-` if that list already exists.
- List `add` (strategic merge): a new `name` adds an item, placed first.
- Check your work with `kubectl kustomize <dir>`.
- Next: [Update](update.md), [Remove](remove.md).
