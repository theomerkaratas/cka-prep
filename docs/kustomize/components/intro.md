# Components: Intro

- **Official Documentation:** [Kustomize Components](https://kubectl.docs.kubernetes.io/guides/config_management/components/)

A **component** is a reusable set of config for one optional feature (for example, a database or a cache). You write it once, then turn it on in only the overlays that need it.

| Where | What goes in it | Example |
| :--- | :--- | :--- |
| **base** | Config every overlay needs | The API Deployment |
| **components** | Config for one optional feature | Postgres, Redis |
| **overlays** | Pick which components to turn on | `dev`, `premium`, `standalone` |

Without components, you would copy the same YAML into every overlay that needs the feature. Copies drift apart over time. A component is written once and included by reference.

---

## Component vs Base vs Resources

- A **base** is always included.
- A **component** is included only when an overlay lists it under `components:`.
- A **component** must declare `kind: Component`. A normal `kustomization.yaml` (a base or an overlay) uses `kind: Kustomization`, which is optional.

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
```

---

## Folder Structure

```text
k8s/
├── base/
│   ├── kustomization.yaml
│   └── api-depl.yaml
├── components/
│   ├── caching/
│   │   ├── kustomization.yaml
│   │   ├── redis-depl.yaml
│   │   └── deployment-patch.yaml
│   └── db/
│       ├── kustomization.yaml
│       ├── postgres-depl.yaml
│       └── deployment-patch.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── premium/
    │   └── kustomization.yaml
    └── standalone/
        └── kustomization.yaml
```

Which overlay uses what:

| Overlay | Base | `db` component | `caching` component |
| :--- | :---: | :---: | :---: |
| **dev** | yes | yes | no |
| **premium** | yes | yes | yes |
| **standalone** | yes | no | yes |

---

## Base

Shared config. Included in every overlay.

```yaml
# k8s/base/kustomization.yaml
resources:
- api-depl.yaml
```

```yaml
# k8s/base/api-depl.yaml
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

## Add a Component to an Overlay

See [Component kustomization.yaml](kustomization-yaml.md) for how to declare a component's resources and patches.

An overlay enables the component under `components:`:
```yaml
# k8s/overlays/dev/kustomization.yaml
resources:
- ../../base
components:
- ../../components/auth
```

Build the overlay to render the base together with the component:

```bash
kustomize build k8s/overlays/dev
```

---

## CKA Quick Reference

- Components are reusable, optional features. Write them once, turn them on per overlay.
- An overlay turns components on with `components:`, not `resources:`.
- Next: [Component kustomization.yaml](kustomization-yaml.md), [Definitions](definitions.md), [Overlays](overlays.md).
