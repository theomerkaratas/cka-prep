# Overlays: Simple

- **Official Documentation:** [Kustomize Bases and Overlays](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#overlay)

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

## Dev Overlay

Dev uses `replicas: 2`.

The `resources` line points up two folders to `base/`. `../` means "go up one folder."

```yaml
# k8s/overlays/dev/kustomization.yaml
resources:
- ../../base

patches:
- target:
    kind: Deployment
    name: nginx-deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 2
```

Result YAML (`kubectl kustomize k8s/overlays/dev`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

---

## Prod Overlay

Prod uses `replicas: 3`, and it also adds a new Grafana Deployment that exists only in prod.

```yaml
# k8s/overlays/prod/kustomization.yaml
resources:
- ../../base
- grafana-deployment.yaml

patches:
- target:
    kind: Deployment
    name: nginx-deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 3
```

```yaml
# k8s/overlays/prod/grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana
```

Result YAML (`kubectl kustomize k8s/overlays/prod`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - image: grafana/grafana
        name: grafana
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

The `---` line separates two objects in one output. Grafana appears only here because only prod adds it.

## Deploy an Overlay

Point `kubectl` at the overlay folder, not at the base:

```bash
kubectl apply -k k8s/overlays/dev
```

```bash
kubectl apply -k k8s/overlays/prod
```

---

## CKA Quick Reference

- Patches in an overlay change the base copy only for that environment.
- An overlay can add new resources by listing them under `resources:`.
- Apply an overlay with `kubectl apply -k <overlay-dir>`.
- Check your work with `kubectl kustomize <overlay-dir>`.
- Next: [Complex](complex.md).
