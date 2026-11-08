# Different Types of Patches

- **Official Documentation:** [Kustomize Patches Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)

Kustomize has **two patch types** and **two places** to write them.

| Patch type | Written as |
| :--- | :--- |
| **JSON 6902** | A list of `op`, `path`, `value` steps. |
| **Strategic merge** | A partial copy of the resource. Kustomize merges it in. |

| Place | Where the patch lives |
| :--- | :--- |
| **Inline** | Inside `kustomization.yaml`. |
| **Separate file** | In its own YAML file, referenced by `path:`. |

Use inline for a few patches. Move them to a file when `kustomization.yaml` gets crowded. Both types and both places work, so pick what you find easier. You can mix them.

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

## 1. JSON 6902: Inline

The target and the patch are both in `kustomization.yaml`.

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

Result: `replicas: 5`

---

## 2. JSON 6902: Separate File

The **target** stays in `kustomization.yaml`. The **list of steps** moves to its own file.

```yaml
# api/replica-patch.yaml
- op: replace
  path: /spec/replicas
  value: 5
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- target:
    kind: Deployment
    name: api-deployment
  path: replica-patch.yaml
```

Result: `replicas: 5`

The file can hold several steps. Each one starts with `- op:`.

---

## 3. Strategic Merge: Inline

Write the part of the Deployment you want to change, like a normal Kubernetes file. Keep `metadata.name` so Kustomize knows which object to change.

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: api-deployment
    spec:
      replicas: 5
```

Result: `replicas: 5`

No `target:` is needed here. The `kind` and `name` in the patch do the matching.

---

## 4. Strategic Merge: Separate File

Same idea, but the patch lives in its own file.

```yaml
# api/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 5
```

```yaml
# api/kustomization.yaml
resources:
- deployment.yaml

patches:
- path: replicas-patch.yaml
```

Result: `replicas: 5`

---

## Comparing the Two Types

| | JSON 6902 | Strategic merge |
| :--- | :--- | :--- |
| **Looks like** | `op`, `path`, `value` steps | Normal Kubernetes YAML |
| **Needs a target?** | Yes (`target:`) | No. Uses `kind` and `name` in the patch. |
| **Can add, remove, replace?** | Yes | Mainly merges and replaces fields |
| **Good for** | One precise change | Several fields at once |

---

## CKA Quick Reference

- JSON 6902 needs a `target:`. Strategic merge matches by `kind` and `metadata.name`.
- Separate files: use `path:` under `patches:`.
- A JSON 6902 patch file is a list of `op` steps.
- A strategic merge patch file is a partial copy of the resource.
- Check your work with `kubectl kustomize <dir>`.
