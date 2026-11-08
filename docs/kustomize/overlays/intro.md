# Overlays: Intro

- **Official Documentation:** [Kustomize Bases and Overlays](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#overlay)

An **overlay** lets you use one set of shared YAML for every environment, and change only what differs per environment.

- **base/**: the shared config (the defaults).
- **overlays/dev/**, **overlays/prod/**: one folder per environment. Each one imports the base and adds patches.

The base is never edited. Each environment gets its own result.

---

## Folder Structure

```text
k8s/
├── base/
│   ├── kustomization.yaml
│   └── nginx-deployment.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── grafana-deployment.yaml
```

---

## Base

The base holds the shared config. Here the Deployment has `replicas: 1`.

```yaml
# k8s/base/nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

```yaml
# k8s/base/kustomization.yaml
resources:
- nginx-deployment.yaml
```

---

## Build, Preview, and Apply an Overlay

Replace `<env_name>` with the overlay directory name, such as `dev` or `prod`.
Build the overlay with the standalone Kustomize CLI:

```bash
kustomize build k8s/overlays/<env_name>
```

Preview the rendered manifests with `kubectl` without applying them:

```bash
kubectl kustomize k8s/overlays/<env_name>
```

Apply the overlay to the cluster:

```bash
kubectl apply -k k8s/overlays/<env_name>
```

## CKA Quick Reference

- `base/` = shared config. `overlays/<env>/` = per-environment patches and extras.
- In an overlay, import the base with `- ../../base` (from `overlays/<env>/`).
- Use `resources:`, not `bases:`.
- Next: [Simple](simple.md), [Complex](complex.md).
