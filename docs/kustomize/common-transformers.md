# Common Transformers

- **Official Documentation:** [Kustomize Built-in Transformers](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/)

Transformers change your resources **without editing the YAML files**. You add them to `kustomization.yaml`, and Kustomize applies them to every resource listed in `resources`.

---

## The Problem

Say you want every resource to have the same label, or a `-dev` suffix on its name. Editing each file by hand does not scale. Common transformers do it in one place.

---

## Example Setup

Directory:

```text
app/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

```yaml
# app/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
```

```yaml
# app/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
```

---

## The Transformers

| Field | What it does | Example value |
| :--- | :--- | :--- |
| **`namespace`** | Puts all resources in a namespace. | `dev` |
| **`namePrefix`** | Adds text to the start of each name. | `team-` |
| **`nameSuffix`** | Adds text to the end of each name. | `-dev` |
| **`labels`** | Adds labels to all resources. | `env: dev` |
| **`commonAnnotations`** | Adds annotations to all resources. | `owner: cka` |

---

## Example: All Transformers Together

```yaml
# app/kustomization.yaml
resources:
- deployment.yaml
- service.yaml

namespace: dev

namePrefix: team-
nameSuffix: -v1

labels:
- pairs:
    env: dev
  includeSelectors: false

commonAnnotations:
  owner: cka
```

Result: the Deployment and Service are renamed `team-web-v1` and placed in the `dev` namespace. Both get `env: dev` as a label and `owner: cka` as an annotation.

Run `kubectl kustomize app/` to see the output.

---

## Notes

- **`includeSelectors: false`** adds the label only to metadata. Without it, the label also goes into selectors, which can break a Deployment when you change labels later. Use `includeSelectors: true` only when you want selectors updated too.
- **`namespace`** sets the namespace for resources. It does not create the namespace. Create it first with `kubectl create namespace dev`.
- **`commonLabels`** is the older form of `labels`. Older guides use it. Use `labels` with `pairs`, as shown above.
- **`commonAnnotations`** is the field the `kubectl` kustomize (v5.8.1) accepts for a simple annotation map. It also adds the annotation to the Pod template.

---

## CKA Quick Reference

- Transformers go in `kustomization.yaml`, not in the resource files.
- They apply to resources listed in that file's `resources` (and any folders it includes).
- `kubectl kustomize <dir>` shows the result before you apply.
- `kubectl apply -k <dir>` applies the transformed resources.
