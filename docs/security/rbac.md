RBAC uses Kubernetes API objects to regulate access to cluster resources based on the roles assigned to individual users, groups, or service accounts.

---

## 1. Creating a Role

A `Role` defines a set of permissions within a **specific namespace**.

### A. Declarative (YAML)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
  # Rule 1: Core API Group (pods)
  - apiGroups: [""] # Empty string represents the Core API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

  # Rule 2: Named API Group (deployments)
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
```

### B. Imperative (CLI)
You can specify multiple rules by repeating the `--verb` and `--resource` flags:

```bash
kubectl create role developer \
  --namespace=development \
  --verb=get,list,watch --resource=pods \
  --verb=get,list --resource=deployments.apps
```

> **Note on syntax:** In the CLI, resources under named groups follow the `<resource>.<group>` pattern (e.g., `deployments.apps`, `ingresses.networking.k8s.io`).

---

## 2. Restricting Access by `resourceNames`

You can restrict permissions to individual instances of a resource by specifying `resourceNames`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: development
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "update", "patch", "delete"]
    resourceNames: ["blue", "orange"]
```

> ⚠️ **Important Caveat:**
> - You **cannot** use `create` or `list` verbs alongside `resourceNames`.
> - **Why?** `create` and `list` operations target collection endpoints (e.g., `POST /api/v1/namespaces/dev/pods`), where object names do not exist yet or are not filtered at the API endpoint level.
> - **Supported verbs:** `get`, `update`, `patch`, `delete`.

---

## 3. Creating a RoleBinding

A `RoleBinding` grants the permissions defined in a `Role` to one or more subjects (`User`, `Group`, or `ServiceAccount`).

### A. Binding to a User

#### Declarative (YAML):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-role-binding
  namespace: development
subjects:
  - kind: User
    name: mahmut
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

#### Imperative (CLI):
```bash
kubectl create rolebinding developer-role-binding \
  --namespace=development \
  --role=developer \
  --user=mahmut
```

---

### B. Binding to a Group

#### Declarative (YAML):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-group-binding
  namespace: development
subjects:
  - kind: Group
    name: frontend-developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

#### Imperative (CLI):
```bash
kubectl create rolebinding dev-group-binding \
  --namespace=development \
  --role=developer \
  --group=frontend-developers
```

---

### C. Binding to a ServiceAccount

#### Declarative (YAML):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sa-binding
  namespace: development
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: development # ServiceAccount namespace is required
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

#### Imperative (CLI):
```bash
kubectl create rolebinding sa-binding \
  --namespace=development \
  --role=developer \
  --serviceaccount=development:my-app-sa
```

---

## 4. Inspection & Verification Commands

### Inspecting Roles and Bindings
```bash
# List roles and bindings
kubectl get roles -n development
kubectl get rolebindings -n development

# Inspect details
kubectl describe role <role-name> -n development
kubectl describe rolebinding <rolebinding-name> -n development
```

### Checking Permissions (`kubectl auth can-i`)
```bash
# Check your own current permissions:
kubectl auth can-i <verb> <resource>

# Impersonate another user (requires admin access):
kubectl auth can-i <verb> <resource> --as=<user-name>

# Specify a target namespace:
kubectl auth can-i <verb> <resource> --as=<user-name> --namespace=<namespace-name>

# Impersonate a ServiceAccount:
kubectl auth can-i get pods --as=system:serviceaccount:development:my-app-sa -n development

# List ALL permissions granted to a user in a namespace:
kubectl auth can-i --list --as=<user-name> --namespace=<namespace-name>


# Also we can execute command like that to see permissions
kubectl get pods --as <user-name>
```