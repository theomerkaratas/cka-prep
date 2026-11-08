# Kustomize Installation

- **Install Guide:** [Kustomize Installation](https://kubectl.docs.kubernetes.io/installation/kustomize/)

---

## Requirements

- A running Kubernetes cluster.
- `kubectl` installed and configured to reach the cluster.

---

## Option 1: Use kubectl (Exam Default)

Kustomize is **built into kubectl**. No install needed.

```bash
kubectl kustomize <dir>
kubectl apply -k <dir>
```

> **Note:** The built-in version can be older than the standalone binary. It is enough for the CKA.

---

## Option 2: Standalone Binary

```bash
# macOS (Homebrew)
brew install kustomize

# Linux / macOS (official install script)
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

---

## Verify

```bash
kustomize version
```

If the command is not found, open a new terminal session and try again. If it still fails, rerun the install script.

---

## CKA Quick Reference

- On the exam, use `kubectl kustomize` and `kubectl apply -k`. Standalone `kustomize` is not required.
