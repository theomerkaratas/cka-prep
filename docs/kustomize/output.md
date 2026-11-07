# Kustomize Output

> **`kustomize build` does NOT apply anything.** It only prints the final manifests to the terminal. Nothing is created in the cluster until you run `kubectl apply`.

---

## Preview (No Changes to Cluster)

```bash
kubectl kustomize k8s/
```

It renders the final Kubernetes manifests from the k8s directory to stdout without applying them to the cluster.

Check with `kubectl get all`: nothing new appears.

---

## Apply

Three equivalent ways:

```bash
# 1. Native kubectl
kubectl apply -k k8s/

# 2. Pipe the build output into apply
kustomize build k8s/ | kubectl apply -f -

# 3. Pipe kubectl kustomize into apply
kubectl kustomize k8s/ | kubectl apply -f -
```

The `|` (pipe) sends the rendered YAML from the left command into `kubectl apply -f -` on the right.

---

## Delete

Same as apply, with `delete`:

```bash
# 1. Native kubectl
kubectl delete -k k8s/

# 2. Pipe the delete
kustomize build k8s/ | kubectş delete -f -
```

---

## CKA Quick Reference

- Preview: `kubectl kustomize <dir>`
- Create/update: `kubectl apply -k <dir>`
- Remove: `kubectl delete -k <dir>`
