# kustomization.yaml File

- **Official Documentation:** [kustomization.yaml Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/)

Kustomize does not read your YAML files directly. It reads a file named **`kustomization.yaml`** in the directory you point it at.

---

## Two Sections

| Section | Purpose |
| :--- | :--- |
| **`resources`** | List of the manifests Kustomize should manage. |
| **Transformers** | Changes to apply to those resources (e.g. labels, image tags, replicas). |

---

## apiVersion and kind

Like other Kubernetes files, you can set `apiVersion` and `kind`. They are optional, but hardcode them so future breaking changes don't affect you:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
```

---

## Example

Directory:

```text
k8s/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

```yaml
# k8s/kustomization.yaml
resources:
- deployment.yaml
- service.yaml

labels:
- pairs:
    company: CodeCloud
```

The `labels` transformer adds `company: CodeCloud` to every resource.

---

## CKA Quick Reference

- The file must be named exactly `kustomization.yaml` (or `kustomization.yml`, or `Kustomization`).
- To render and apply, see [output.md](output.md).
