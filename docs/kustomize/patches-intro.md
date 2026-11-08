# Patches Intro

- **Official Documentation:** [Kustomize Patches Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)

Patches change **one specific object** (or a few). Common transformers change **everything**.

| Use this | When |
| :--- | :--- |
| **Common transformer** | Same change on all resources (a label, a namespace). |
| **Patch** | Change one object (e.g. `replicas` on one Deployment). |

---

## What a Patch Needs

A patch has three parts:

1. **Operation:** what to do (`add`, `remove`, or `replace`).
2. **Target:** which object to change.
3. **Value:** the new value. Needed for `add` and `replace`, not for `remove`.

### Operations

| Operation | Meaning | Example |
| :--- | :--- | :--- |
| **`add`** | Add something new. | Add a container to a list of containers. |
| **`remove`** | Delete something. | Remove a label. |
| **`replace`** | Swap a value for another. | Change replicas from `5` to `10`. |

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
    spec:
      containers:
      - name: api
        image: nginx
```

---

## Example 1: Change the Name

Goal: rename `api-deployment` to `web-deployment`.

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
      path: /metadata/name
      value: web-deployment
```

Result: `metadata.name: web-deployment`

---

## Example 2: Change Replicas

Goal: change `replicas` from `1` to `5`.

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

## Example 3: Add a Label

Goal: add the label `team: platform` to the Deployment.

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
      path: /metadata/labels/team
      value: platform
```

Result: `metadata.labels` now has `team: platform`.

---

## Example 4: Remove a Label

Goal: remove the label `tier` from the Deployment. Assume it was `tier: backend`.

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
      path: /metadata/labels/tier
```

Result: the `tier` label is gone. No `value` is needed.

---

## Example 5: Add a Container

Goal: add a second container called `sidecar` to the Pod template.

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
        image: busybox
```

Result: the Deployment now has two containers, `api` and `sidecar`.

The `-` at the end of the path means "add to the end of the list."

---

## Example 6: Change a Container Image

Goal: change the image of the first container (`containers/0`) to `haproxy:2.4`.

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

Result: `image: haproxy:2.4`

Lists start at `0`. So `containers/0` is the first container.

---

## Example 7: Match by Label Instead of Name

Goal: change replicas on every Deployment with the label `app: api`.

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- target:
    kind: Deployment
    labelSelector: "app=api"
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 3
```

Result: `replicas: 3` on each matching Deployment.

Use a label selector when you don't know the exact name, or when you want to match several objects.

---

## The `path`

`path` is the route to the field, written with `/`.

| Field in YAML | `path` |
| :--- | :--- |
| `metadata.name` | `/metadata/name` |
| `spec.replicas` | `/spec/replicas` |

If the field is deeper, keep adding `/` levels.

---

## The `|-` Symbol

`patch: |-` means the patch is written as text right below it. The `-` at the start of each line starts a list item. Keep the indentation the same.

---

## CKA Quick Reference

- Patches are for **one** object. Common transformers are for **all** objects.
- Each patch needs an operation, a target, and (for `add`/`replace`) a value.
- `path` is the field location, like `/spec/replicas`.
- Check your work with `kubectl kustomize <dir>`.
- Next: [Different Patch Types](different-types-of-patches.md).
