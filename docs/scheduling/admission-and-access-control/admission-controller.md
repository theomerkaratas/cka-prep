# Admission Controllers

An **Admission Controller** is a piece of code that intercepts requests to the Kubernetes API server **prior to persistence of the object**, but **after** the request is authenticated and authorized.

---

## API Request Lifecycle

```text
  [ Client Request ]
          │
          ▼
   1. Authentication     (Who are you? e.g. X.509 certs, tokens)
          │
          ▼
   2. Authorization      (Are you allowed to do this? e.g. RBAC)
          │
          ▼
   3. Admission Control  (Can this request be modified or admitted? Mutating & Validating)
          │
          ▼
    [ Saved to etcd ]
```

> **Important:** Admission controllers only intercept requests that create, modify, or delete resources (`CREATE`, `UPDATE`, `DELETE`, `CONNECT`). They **do not** intercept read requests (`GET`, `LIST`, `WATCH`).

---

## Built-in Admission Plugins

Kubernetes comes with numerous pre-compiled admission plugins:

- **`NamespaceLifecycle`:** Prevents creating resources in namespaces that are terminating, or in non-existent namespaces.
- **`LimitRanger`:** Enforces `LimitRange` constraints and injects default requests/limits into pods.
- **`ResourceQuota`:** Enforces namespace quotas and rejects requests that exceed quotas.
- **`NodeRestriction`:** Restricts what resources a kubelet can modify (e.g. only its own Node and Pods).
- **`DefaultStorageClass`:** Automatically assigns a default StorageClass to PVCs that do not specify one.
- **`AlwaysPullImages`:** Modifies new pods to force the image pull policy to `Always`.
- **`NamespaceAutoProvision`:** *(Disabled by default)* Automatically creates a namespace if it does not exist when a resource is submitted.

---

## Viewing Enabled Admission Plugins

### Method 1: Check Kube-Apiserver Help
```bash
kube-apiserver -h | grep enable-admission-plugins
```

### Method 2: Inspect Kube-Apiserver Manifest (kubeadm)
Check `/etc/kubernetes/manifests/kube-apiserver.yaml`:
```bash
grep -i admission-plugins /etc/kubernetes/manifests/kube-apiserver.yaml
```

### Method 3: Inspect Running Process
```bash
ps aux | grep kube-apiserver | grep enable-admission-plugins
```

---

## Enabling and Disabling Plugins

To modify admission plugins on a `kubeadm` cluster, edit `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --enable-admission-plugins=NodeRestriction,NamespaceLifecycle,LimitRanger
    - --disable-admission-plugins=DefaultStorageClass
```

Because `kube-apiserver` runs as a static pod, the `kubelet` will automatically restart it whenever you save changes to this manifest.


