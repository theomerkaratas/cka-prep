# Managing Directories

- **Official Documentation:** [Kustomize Reference](https://kubectl.docs.kubernetes.io/references/kustomize/)

As your manifests grow, you will organize them into subdirectories. Kustomize lets you deploy a whole tree of folders with **one command**.

---

## The Rule

Kustomize can reference a **directory** in `resources`, but only if that directory contains its own `kustomization.yaml`.

| Where | What it lists |
| :--- | :--- |
| **Subdirectory** `k8s/api/` | The YAML files in `api/` |
| **Root** `k8s/` | The **subdirectories** (`api`, `database`) |

Each level only knows about the level directly below it.

---

## Why One File per Folder?

You could list every file in one root `kustomization.yaml`. Splitting it up has benefits:

- **Shorter files:** A long list of paths is hard to read. Each folder's file stays small.
- **Clear ownership:** Each folder lists its own files. Changes stay in that folder.
- **Reuse:** A folder with its own `kustomization.yaml` can be reused elsewhere, for example in an overlay.
- **Easy testing:** Run `kubectl kustomize api/` to check one folder on its own.

For a small exercise, one file is fine.

---

## Example Layout

```text
k8s/
├── kustomization.yaml     # root: lists the folders
├── api/
│   ├── kustomization.yaml # lists the api files
│   ├── deployment.yaml
│   └── service.yaml
└── database/
    ├── kustomization.yaml # lists the database files
    ├── deployment.yaml
    └── service.yaml
```

---

## Step 1: Subdirectory `kustomization.yaml` Files

List only the files in that folder.

```yaml
# k8s/api/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
```

```yaml
# k8s/database/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
```

---

## Step 2: Root `kustomization.yaml`

List the **directories**, not the files.

```yaml
# k8s/kustomization.yaml
resources:
- api
- database
```

Kustomize goes into `api/` and `database/`, reads their `kustomization.yaml` files, and collects all their resources.

---

## Apply and Delete

Run these from anywhere, pointing at the root directory:

```bash
kubectl apply -k k8s/
```

```bash
kubectl delete -k k8s/
```

To see the final YAML before applying:

```bash
kubectl kustomize k8s/
```

---

## Nested Directories

Folders can be nested. Each folder just needs a `kustomization.yaml` that points to its children.

```text
k8s/
├── kustomization.yaml     # resources: [apps]
└── apps/
    ├── kustomization.yaml # resources: [api, database]
    ├── api/
    │   ├── kustomization.yaml
    │   └── deployment.yaml
    └── database/
        ├── kustomization.yaml
        └── deployment.yaml
```

```yaml
# k8s/kustomization.yaml
resources:
- apps
```

```yaml
# k8s/apps/kustomization.yaml
resources:
- api
- database
```

---

## CKA Quick Reference

- Each directory in `resources` **must** contain a `kustomization.yaml`. Otherwise you get an error.
- In the root file, list **folders**. In a subdirectory file, list **files**.
- `kubectl apply -k <dir>` applies everything under that directory.
- `kubectl kustomize <dir>` prints the output without applying it. Use it to check your work.
- Paths are relative to the `kustomization.yaml` that contains them.
