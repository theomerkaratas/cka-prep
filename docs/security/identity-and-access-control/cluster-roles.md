# Kubernetes ClusterRole and ClusterRoleBinding Guide

While `Role` and `RoleBinding` are restricted to a specific namespace, `ClusterRole` and `ClusterRoleBinding` are **cluster-scoped** objects used to manage permissions across the entire cluster.

---

## 1. Key Concepts

- **Cluster-Scoped Resources:** Resources that do not belong to any namespace (e.g., `nodes`, `persistentvolumes`, `namespaces`, `storageclasses`) can **only** be authorized using a `ClusterRole`.
- **`Namespaced` Resources Across All Namespaces:** You can define a `ClusterRole` for `namespaced` resources (e.g., `pods`, `deployments`, `secrets`). When bound via a `ClusterRoleBinding`, the subject gets access to those resources across **all** namespaces in the cluster.
- **No `namespace` field:** Neither `ClusterRole` nor `ClusterRoleBinding` definitions have a `metadata.namespace` field.

### Checking Resource Scope
Use `kubectl api-resources` to quickly identify which resources are cluster-wide versus `namespaced`:

```bash
# List all namespaced resources (Pods, Deployments, Secrets, etc.)
kubectl api-resources --namespaced=true

# List all cluster-scoped resources (Nodes, PVs, StorageClasses, etc.)
kubectl api-resources --namespaced=false
```

---

## 2. Creating a ClusterRole

### A. Declarative (YAML)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-admin # Notice: No namespace field here
rules:
  # Cluster-scoped resources (Core API group)
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch", "create", "delete"]

  # Can also include namespaced resources to grant access across all namespaces
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch"]
```

### B. Imperative (CLI)
```bash
# Basic ClusterRole for a single resource:
kubectl create clusterrole node-admin \
  --verb=get,list,watch,create,delete \
  --resource=nodes

# Multiple rules / multiple resources:
kubectl create clusterrole storage-admin \
  --verb=get,list,watch --resource=persistentvolumes \
  --verb=get,list --resource=storageclasses.storage.k8s.io

# Generate YAML template without applying:
kubectl create clusterrole node-admin \
  --verb=get,list --resource=nodes \
  --dry-run=client -o yaml > clusterrole.yaml
```

---

## 3. Creating a ClusterRoleBinding

A `ClusterRoleBinding` grants the permissions in a `ClusterRole` cluster-wide to users, groups, or service accounts.

### A. Binding to a User

#### Declarative (YAML):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-admin-binding
subjects:
  - kind: User
    name: devops-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-admin
  apiGroup: rbac.authorization.k8s.io
```

#### Imperative (CLI):
```bash
kubectl create clusterrolebinding node-admin-binding \
  --clusterrole=node-admin \
  --user=devops-user
```

---

### B. Binding to a Group

#### Declarative (YAML):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-readers-binding
subjects:
  - kind: Group
    name: platform-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view # Built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

#### Imperative (CLI):
```bash
kubectl create clusterrolebinding cluster-readers-binding \
  --clusterrole=view \
  --group=platform-team
```

---

### C. Binding to a ServiceAccount

#### Declarative (YAML):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-sa-binding
subjects:
  - kind: ServiceAccount
    name: prometheus-sa
    namespace: monitoring # Namespace of the ServiceAccount is required
roleRef:
  kind: ClusterRole
  name: node-admin
  apiGroup: rbac.authorization.k8s.io
```

#### Imperative (CLI):
```bash
kubectl create clusterrolebinding monitoring-sa-binding \
  --clusterrole=node-admin \
  --serviceaccount=monitoring:prometheus-sa
```

---

## 4. Special Pattern: `ClusterRole` + `RoleBinding` (Reusable Roles)

You do **not** have to bind a `ClusterRole` with a `ClusterRoleBinding`. You can bind a `ClusterRole` using a regular namespaced **`RoleBinding`**.

- **What happens?** The user inherits the permissions defined in the `ClusterRole`, but **only within that specific namespace**.
- **Why use it?** Avoid duplicating identical `Role` definitions across 50 different namespaces. Define common roles (e.g., `pod-reader`) once as a `ClusterRole`, then attach them locally per namespace.

```bash
# Grant 'john' pod-reader permissions ONLY inside the 'development' namespace:
kubectl create rolebinding dev-pod-reader-binding \
  --clusterrole=pod-reader \
  --user=john \
  --namespace=development
```

---

## 5. Inspection and Verification Commands

### View Existing ClusterRoles and Bindings
```bash
# List all ClusterRoles
kubectl get clusterroles

# List all ClusterRoleBindings
kubectl get clusterrolebindings

# Describe a specific object
kubectl describe clusterrole node-admin
kubectl describe clusterrolebinding node-admin-binding
```

### Testing Cluster-Wide Permissions (`kubectl auth can-i`)
```bash
# Test cluster-scoped permissions for a user:
kubectl auth can-i list nodes --as=devops-user

# Test access across all namespaces:
kubectl auth can-i get pods --all-namespaces --as=devops-user

# Test for a ServiceAccount:
kubectl auth can-i list nodes --as=system:serviceaccount:monitoring:prometheus-sa

# List all global permissions assigned to a user:
kubectl auth can-i --list --as=devops-user
```


### TIP!
- We can count the result via `| wc -l`
- **`get`**: Retrieves a **single, specific** resource by its name.  
  *Example:* `kubectl get node worker-1`  
  *HTTP:* `GET /api/v1/nodes/worker-1`

- **`list`**: Retrieves a **collection / all** resources of that type.  
  *Example:* `kubectl get nodes`  
  *HTTP:* `GET /api/v1/nodes`

> **Key Rule:** With only `get`, you cannot run `kubectl get nodes`. With only `list`, you cannot target a specific node by name. That is why they are almost always granted together (`verbs: ["get", "list"]`).