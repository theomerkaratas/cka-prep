# Helm 2 vs Helm 3

For the CKA exam, you must know that **Helm 3 is the current standard** and Helm 2 is deprecated.

| Feature             | Helm 2                       | Helm 3                             |
| :------------------ | :--------------------------- | :--------------------------------- |
| **Architecture**    | **Client + Server** (Tiller) | **Client-only** (Direct API calls) |
| **Tiller**          | Required in cluster          | **Removed** (deleted for security) |
| **Release Storage** | Secrets + ConfigMaps         | **Secrets only**                   |
| **Namespace Scope** | Single namespace (or all)    | Full namespace support             |

# Helm 2 and Helm 3 Quick Reference

For the CKA exam, you must know that **Helm 3 is the current standard** and Helm 2 is deprecated.

| Feature             | Helm 2                       | Helm 3                             |
| :------------------ | :--------------------------- | :--------------------------------- |
| **Architecture**    | **Client + Server** (Tiller) | **Client-only** (Direct API calls) |
| **Tiller**          | Required in cluster          | **Removed** (deleted for security) |
| **Release Storage** | Secrets + ConfigMaps         | **Secrets only**                   |
| **Namespace Scope** | Single namespace (or all)    | Full namespace support             |

---

## Key Differences in Practice

### 1. No More Tiller (Security Improvement)

- **Helm 2:** Required installing a service called `Tiller` inside the cluster. This was a security risk (required cluster-admin rights).
- **Helm 3:** Uses your existing `kubectl` credentials. It talks directly to the Kubernetes API. **No Tiller installation needed**.

### 2. Better Namespace Support

- **Helm 2:** Awkward to manage releases in different namespaces.
- **Helm 3:** `helm list`, `helm install`, `helm upgrade` all support the `-n` or `--namespace` flag naturally.

### 3. Simplified Storage

- **Helm 2:** Kept release info in both Secrets and ConfigMaps (messy).
- **Helm 3:** Stores release information **only in Secrets**. This makes cleanup easier.

---

## Upgrade/Migrate from Helm 2 → Helm 3

Although you likely won't do this in the exam, it's good to know:

```bash
# Install the upgrade plugin
helm plugin install https://github.com/helm/helm-2to3

# Run the migration
helm 2to3 migrate

# Verify (optional)
helm 2to3 status

# Remove Helm 2
helm 2to3 cleanup
```

---

## Exam Tips

- **Always use Helm 3 commands.** Do not worry about Helm 2 syntax in the exam.
- **Check the environment:** If the exam question explicitly mentions `Tiller`, it is a trick question referring to Helm 2. Use Helm 3 commands anyway unless told otherwise.
- **Check version:** `helm version` will show you `v2` or `v3`.
