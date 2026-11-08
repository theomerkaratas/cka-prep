# Kustomize vs Helm

- **Helm Docs:** [Helm Documentation](https://helm.sh/docs/)
- **Kustomize Docs:** [Kustomize Docs](https://kubectl.docs.kubernetes.io/references/kustomize/)

Both tools solve the same problem: **one app, different settings per environment**. They do it differently.

---

## Quick Comparison

| | Kustomize | Helm |
| :--- | :--- | :--- |
| **Approach** | Patch plain YAML (base + overlays) | Templates + values (`values.yaml`) |
| **Syntax** | Plain YAML, no templating | Go templates `{{ .Values.x }}` |
| **Valid YAML?** | Yes | No (templates must be rendered first) |
| **Built into kubectl?** | Yes (`kubectl apply -k`) | No, separate binary |
| **Packaging / versioning** | No | Yes (charts, releases, rollback) |
| **Logic (if/loops/functions)** | No | Yes |

---

## Helm in One Example

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  containers:
  - image: nginx:{{ .Values.image.tag }}
```

```yaml
# values.yaml
replicaCount: 1
image:
  tag: 2.4.4
```

Per environment, Helm uses a separate values file: `values.dev.yaml`, `values.prod.yaml`.

---

## When to Use Which

- **Kustomize:** simple per-environment changes, and you want readable YAML.
- **Helm:** you need packaging, releases, rollbacks, or logic in templates, or you're installing third-party apps.

---

## CKA Quick Reference

- Kustomize: `kubectl apply -k <dir>`, `kubectl kustomize <dir>` to preview.
- Helm: `helm install`, `helm upgrade`, `helm rollback`, `helm list`.
- Helm templates are not valid YAML, so you can't `kubectl apply` them directly.
