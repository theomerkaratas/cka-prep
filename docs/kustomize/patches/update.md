# Patches: Update

- **Official Documentation:** [Kustomize Patches Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)

**Update** changes a value that already exists. In JSON 6902 this is the `replace` operation. The object's key or list item must already be there.

| Goal | JSON 6902 | Strategic merge |
| :--- | :--- | :--- |
| Change a dictionary value | `op: replace` on `.../labels/<key>` | Write the key with the new value |
| Change a list item | `op: replace` on `.../containers/0/<field>` | Write the item with its `name` |

- JSON 6902 `replace` needs a `value`.
- List items in JSON 6902 are numbered from `0`.
- Strategic merge matches list items by **`name`**, not by position.

For the setup, see [Patches Intro](intro.md#example-setup).

---

## Update a Key (Dictionary)

Goal: change `component: api` to `component: web` in the Pod template labels.

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
    - op: replace
      path: /spec/template/metadata/labels/component
      value: web
```

Result: `component: web`

The path goes to the dictionary, then to the key name: `/spec/template/metadata/labels` + `/component`.

### 2. Strategic Merge

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

---

## Update a List Item (Container Image)

Goal: change the image of the first container (`api`) to `haproxy:2.4`.

### 1. JSON 6902

The first container is at index `0`.

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

Result: `api` now uses `haproxy:2.4`. `logger` is untouched.

### 2. Strategic Merge

Only the `name` and the changed field are needed.

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

Result YAML (`kubectl kustomize api/`) for either type:

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
      - image: haproxy:2.4
        name: api
      - image: busybox
        name: logger
```

**Gotcha:** JSON 6902 uses the **index** (`containers/0`). If the containers are reordered, the index points to a different container. Strategic merge uses the **name**, so it is safer when the order can change.

---

## CKA Quick Reference

- JSON 6902 update: `op: replace` with the full `path` and a `value`.
- Dictionary path: `.../labels/<key>`.
- List path (JSON 6902): `.../containers/<index>/<field>`.
- List update (strategic merge): write the item with its `name` and the changed field.
- Check your work with `kubectl kustomize <dir>`.
- Next: [Add](add.md), [Remove](remove.md).
