# Kustomize Introduction

- **Official Documentation:** [Kustomize Docs](https://kubectl.docs.kubernetes.io/references/kustomize/)

Kustomize lets you **reuse one set of Kubernetes YAML** and change only what differs per environment (dev, staging, prod). No templates, just plain YAML.

---

## The Problem

Copying the same manifests into `dev/`, `staging/`, and `prod/` works, but every change must be repeated in every folder. Kustomize avoids the duplication.

---

## Core Concepts

| Concept | What It Is |
| :--- | :--- |
| **Base** | The shared, default resources (e.g. `deployment.yaml`, `service.yaml`). |
| **Overlay** | Per-environment changes applied on top of the base (e.g. `replicas: 5` for prod). |
| **kustomization.yaml** | The file that lists resources and patches. Every base and overlay has one. |

---

## Folder Structure

```text
app/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

---

## Minimal Example

```yaml
# base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
```

```yaml
# overlays/prod/kustomization.yaml
resources:
- ../../base
replicas:
- name: nginx-deployment
  count: 5
```

---

## Commands

```bash
# Build the final manifests (prints to stdout)
kubectl kustomize overlays/prod

# Build and apply to the cluster
kubectl apply -k overlays/prod

# Delete everything built from the overlay
kubectl delete -k overlays/prod
```

> **Note:** `kubectl kustomize` renders only. `kubectl apply -k` renders and applies. Use `-k` (not `-f`) to point at a folder with a `kustomization.yaml`.

---

## CKA Quick Reference

- Kustomize is **built into kubectl**, so no extra install is needed for the exam.
- Look for a `kustomization.yaml` in a folder before writing YAML from scratch.
- Use `kubectl kustomize <dir>` to preview output before applying.
- Overlays override base values by matching the same `name` and `kind`.
