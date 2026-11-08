# Patches Intro

- **Official Documentation:** [Kustomize Patches Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)

Patches change **one specific object** (or a few). Common transformers change **everything**.

| Use this | When |
| :--- | :--- |
| **Common transformer** | Same change on all resources (a label, a namespace). |
| **Patch** | Change one object (e.g. `replicas` on one Deployment). |

---

## Pick the Operation

Every patch does one of three things. Each has its own page:

| Operation | Meaning | Page |
| :--- | :--- | :--- |
| **`add`** | Add a new key or a new list item. | [Add](add.md) |
| **`replace`** (update) | Change an existing value. | [Update](update.md) |
| **`remove`** | Delete a key or a list item. | [Remove](remove.md) |

Each page shows both patch types side by side: **JSON 6902** and **strategic merge**.

---

## What a Patch Needs

A patch has three parts:

1. **Operation:** what to do (`add`, `replace`, or `remove`).
2. **Target:** which object to change.
3. **Value:** the new value. Needed for `add` and `replace`, not for `remove`.

### Target

The target finds the object. You can match on:

- `kind` (for example `Deployment`)
- `version` (for example `apps/v1`)
- `name`
- `namespace`
- `labelSelector`
- `annotationSelector`

You can combine several of these to match exactly one object.

---

## Two Patch Types

Kustomize has **two patch types**. You can write either one inline or in a separate file.

| | JSON 6902 | Strategic merge |
| :--- | :--- | :--- |
| **Looks like** | `op`, `path`, `value` steps | Normal Kubernetes YAML |
| **Needs a target?** | Yes (`target:`) | No. Uses `kind` and `name` in the patch. |
| **Good for** | One precise change | Several fields at once |

Both types and both places (inline or file) work. Pick what you find easier. You can mix them.

- **Inline:** the patch is written under `patch: |-` in `kustomization.yaml`.
- **Separate file:** the patch lives in its own YAML file, referenced by `path:`.

---

## Example Setup

Directory:

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
      - name: logger
        image: busybox
```

The examples on the operation pages all start from this file.

---

## Quick Example

Goal: change `replicas` from `1` to `5`, using a JSON 6902 patch inline.

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
      path: /spec/replicas
      value: 5
```

Result: `spec.replicas: 5`

---

## The `path`

`path` is the route to the field, written with `/`.

| Field in YAML | `path` |
| :--- | :--- |
| `metadata.name` | `/metadata/name` |
| `spec.replicas` | `/spec/replicas` |
| `metadata.labels.app` | `/metadata/labels/app` |
| first container's image | `/spec/template/spec/containers/0/image` |

Lists start at `0`. So `containers/0` is the first container. `/-` means "the end of the list."

---

## The `|-` Symbol

`patch: |-` means the patch is written as text right below it. Keep the indentation the same.

---

## CKA Quick Reference

- Patches are for **one** object. Common transformers are for **all** objects.
- Each patch needs an operation, a target, and (for `add`/`replace`) a value.
- JSON 6902 needs a `target:`. Strategic merge matches by `kind` and `metadata.name`.
- `path` is the field location, like `/spec/replicas`.
- Check your work with `kubectl kustomize <dir>`.
- Next: [Add](add.md), [Update](update.md), [Remove](remove.md).
