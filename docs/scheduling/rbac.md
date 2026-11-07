# Role-Based Access Control (RBAC)

Role-Based Access Control (RBAC) determines whether a user, group, or service account has permission to perform actions on Kubernetes resources.

---

## Core RBAC Primitives

| Component | Scope | Description |
| :--- | :--- | :--- |
| **`Role`** | Namespaced | Defines a set of permissions (verbs) on resources within a specific namespace. |
| **`ClusterRole`** | Cluster-scoped | Defines permissions across the entire cluster (e.g. Nodes, PVs, or all namespaces). |
| **`RoleBinding`** | Namespaced | Grants the permissions defined in a Role (or ClusterRole) to a user, group, or ServiceAccount within a namespace. |
| **`ClusterRoleBinding`** | Cluster-scoped | Grants permissions cluster-wide. |

---

## 1. Defining a Role

A Role defines what actions (`verbs`) can be performed on which `resources` within an `apiGroup`:

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-developer
  namespace: default
rules:
- apiGroups: [""] # Core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch", "create", "delete"]
  resourceNames: ["blue-pod", "green-pod"] # Optional: restrict to specific resource instances
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update"]
```

---

## 2. Binding the Role (RoleBinding)

A `RoleBinding` binds the role to subjects (Users, Groups, or ServiceAccounts):

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: default
subjects:
- kind: User
  name: john
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-developer
  apiGroup: rbac.authorization.k8s.io
```

---

## 3. Imperative Commands (CKA Exam Speed)

Creating RBAC manifests manually takes time. Use imperative commands to create them instantly:

```bash
# Create a Role
kubectl create role dev-role \
  --verb=get,list,create \
  --resource=pods \
  --namespace=default

# Create a RoleBinding
kubectl create rolebinding dev-user-binding \
  --role=dev-role \
  --user=john \
  --namespace=default

# Create a ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create a ClusterRoleBinding
kubectl create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader \
  --user=john
```

---

## 4. Verifying Permissions with `auth can-i`

Test whether a specific user or service account has permissions:

```bash
# Check if current user can delete pods
kubectl auth can-i delete pods

# Check if user 'john' can create pods in namespace 'default'
kubectl auth can-i create pods --as=john --namespace=default

# Check if a ServiceAccount can list secrets
kubectl auth can-i list secrets --as=system:serviceaccount:default:my-sa -n default
```
