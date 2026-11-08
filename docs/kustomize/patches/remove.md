# Patches: Remove

- **Official Documentation:** [Kustomize Patches Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)

**Remove** deletes a key from a dictionary or an item from a list. There are **two ways** to do it, and each patch type has its own:

| Goal | JSON 6902 | Strategic merge |
| :--- | :--- | :--- |
| Remove a dictionary key | `op: remove` on `.../labels/<key>` | Set the key to `null` |
| Remove a list item | `op: remove` on `.../containers/<index>` | Write the item with `$patch: delete` |

- `remove` in JSON 6902 needs **no `value`**.
- Strategic merge has **no `remove` operation**. You describe what should be gone.
- In JSON 6902, list items are numbered from `0`.

For the setup, see [Patches Intro](intro.md#example-setup).

---

## Remove a Key (Dictionary)

Goal: remove the `component` label from the Pod template.

### 1. JSON 6902

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
      path: /spec/template/metadata/labels/component
```

Result: `component` is removed. No `value` is needed.

### 2. Strategic Merge (`null`)

Set the key to `null`.

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
        component: null
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: label-patch.yaml
```

Result: `component` is removed.

---

## Remove a List Item (Container)

Goal: remove the `logger` container.

### 1. JSON 6902

`logger` is at index `1`.

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

### 2. Strategic Merge (`$patch: delete`)

Write the item by its `name` and add `$patch: delete`.

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
      - name: logger
        $patch: delete
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: container-patch.yaml
```

Result: `logger` is removed.

Result YAML (`kubectl kustomize api/`) for the container removal:

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

## Which Way to Remove?

| Situation | Use |
| :--- | :--- |
| You want a precise, index-based change | JSON 6902 `remove` |
| You want to match by name, and the order may change | Strategic merge (`null` or `$patch: delete`) |

---

## CKA Quick Reference

- JSON 6902 remove: `op: remove` and the full `path`. No `value`.
- Dictionary key, strategic merge: set it to `null`.
- List item, strategic merge: `$patch: delete` with the item's `name`.
- JSON 6902 list removal uses the index (`containers/1`).
- Check your work with `kubectl kustomize <dir>`.
- Next: [Add](add.md), [Update](update.md).
