Once a request to the Kubernetes API server is successfully **authenticated** (answering _"Who are you?"_), it immediately enters the **authorization** stage (answering _"Do you have permission to perform this action on this resource?"_).

---

## 1. How Authorization Works in the API Server

Every incoming request contains three key attributes for authorization:

- **User / Identity:** The authenticated user or ServiceAccount (and their groups).
- **Resource / Path:** The target resource (e.g., `pods`, `deployments`) or non-resource URL (e.g., `/metrics`, `/healthz`).
- **Verb:** The action requested (e.g., `get`, `list`, `create`, `update`, `delete`, `watch`).

----------

## 2. The Four Primary Authorization Modes

### A. Node Authorization (`Node`)

A specialized authorizer specifically designed to limit the permissions of **kubelets**.

- **Purpose:** Prevents a compromised worker node from modifying cluster-wide configurations or accessing pods/secrets running on other nodes.
- **How it works:**
    - `Kubelets` belong to the `system:nodes` group with names like `system:node:<nodeName>`.
    - The Node authorizer grants `kubelets` read access only to Secrets, ConfigMaps, and Pods that are scheduled to run on **their own node**, and write access only to their own Node status, Pod status, and Events.
- **Typical usage:** Always enabled alongside RBAC: `--authorization-mode=Node,RBAC`.

---

### B. ABAC (Attribute-Based Access Control)

Authorizes requests based on a set of static policy rules defined in a local JSON file.

- **How it works:**
    - Rules are written to a file on the master node (e.g., `/etc/kubernetes/abac-policy.json`).
    - Passed via `--authorization-policy-file=<path>`.

```json
{
  "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
  "kind": "Policy",
  "spec": {
    "user": "alice",
    "namespace": "production",
    "resource": "pods",
    "readonly": true
  }
}
```
- **Disadvantages:**
    - **Static & Inflexible:** Any policy change requires editing a local file on the control plane node and restarting the API server.
    - Rarely used in modern production clusters; largely superseded by RBAC.

---

### C. RBAC (Role-Based Access Control)

The de facto standard and most widely used authorization mode in Kubernetes.

- **How it works:**
    - Manages permissions entirely using native Kubernetes API objects (`Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding`).
    - No master node restarts needed: rules are updated dynamically using `kubectl` or YAML manifests just like any other resource.
    - Strictly additive: permissions can only be granted, not denied explicitly (allow-list model).
- **Core objects:**
    - **Role / ClusterRole:** Defines _what_ actions (`verbs`) can be taken on _which_ objects (`resources`, `apiGroups`).
    - **RoleBinding / ClusterRoleBinding:** Attaches that Role to users, groups, or ServiceAccounts.
- **Typical usage:** Enabled by default on virtually all modern Kubernetes distributions (`--authorization-mode=Node,RBAC`).

---

### D. Webhook Authorization

Delegates the authorization decision to an external, third-party HTTP service.

- **Purpose:** Allows integrating Kubernetes authorization with external systems, custom business logic, or centralized identity & policy engines (e.g., Open Policy Agent / OPA Gatekeeper).
- **How it works:**
    - Configured with a kubeconfig-style file via `--authorization-webhook-config-file=<path>`.
    - When a request arrives, the API server sends a POST request with a `SubjectAccessReview` JSON payload to the remote HTTP webhook.
    - The external service evaluates the request and responds with a JSON object indicating `allowed: true` or `allowed: false`.

```
                    ┌────────────────────────┐
                    │     kube-apiserver     │
                    └───────────┬────────────┘
                                │
       1. POST SubjectAccessReview
          (User, Resource, Verb)
                                │
                                ▼
                    ┌────────────────────────┐
                    │ External Authz Webhook │
                    │   (e.g., OPA / Custom) │
                    └───────────┬────────────┘
                                │
       2. Response: { "allowed": true }
                                │
                                ▼
                    ┌────────────────────────┐
                    │     kube-apiserver     │
                    │   (Applies Decision)   │
                    └────────────────────────┘
```
---

## 3. Comparison Summary

|Mode|Configuration Method|Dynamic (No API restart)?|Primary Use Case|
|---|---|---|---|
|**Node**|`--authorization-mode=Node`|Built-in logic|Restricting Kubelet access to its own node resources.|
|**ABAC**|Policy JSON file on control plane|❌ No|Legacy clusters; static, low-maintenance setups.|
|**RBAC**|Native K8s manifests (`kubectl`)|✅ Yes|General-purpose user & workload permissions inside the cluster.|
|**Webhook**|External HTTPS service URL|✅ Yes (external logic)|Integration with external policy engines, corporate SSO, or custom authorization rules.|

------

# The `--authorization-mode` Flag in Kubernetes

The `--authorization-mode` flag defines how the `kube-apiserver` evaluates and authorizes incoming API requests. It accepts an ordered, comma-separated list of authorizers.

## 1. Special Authorizer Modes: `AlwaysAllow` & `AlwaysDeny`

### `AlwaysAllow`

- **Behavior:** Bypasses authorization completely and unconditionally approves **every** request.
- **Security:** Insecure; any caller (even unauthenticated/anonymous) has full cluster privileges.
- **Use Cases:**
    - Local development environments.
    - Emergency disaster recovery (e.g., if RBAC policies are broken and lock out admins).

### `AlwaysDeny`

- **Behavior:** Unconditionally rejects **every** request with `403 Forbidden`, regardless of the user's role or identity.
- **Use Cases:**
    - Purely for testing authorization logic and failure handling within Kubernetes codebase test suites.

> Default is `AlwaysAllow`

---

## 2. All Available Modes at a Glance

|Mode|Behavior|Common Scenario|
|---|---|---|
|**`Node`**|Restricts Kubelets to access only resources on their own node|Enabled by default alongside RBAC|
|**`RBAC`**|Dynamically evaluates `Role` and `ClusterRole` objects|Production standard|
|**`Webhook`**|Forwards authorization requests to an external HTTP service|OPA / custom policy engines|
|**`ABAC`**|Reads static JSON policy files from control plane disk|Legacy / Deprecated|
|**`AlwaysAllow`**|Grants permission to every incoming request|Dev / testing / recovery|
|**`AlwaysDeny`**|Denies permission to every incoming request|Testing authorization failures|

---

## 3. CKA Troubleshooting Scenario

A common CKA troubleshooting task involves an API server that rejects all operations with `403 Forbidden` errors:

1. Check the static pod manifest:

```sh
grep authorization-mode /etc/kubernetes/manifests/kube-apiserver.yaml
```

2. If it is misconfigured (e.g., set to `AlwaysDeny` or missing `RBAC`):

```sh
# Fix it to the production standard:
- --authorization-mode=Node,RBAC
```
3. Save the file; `Kubelet` will automatically restart the `kube-apiserver` static pod.

> If we have multiple modes to configure, our request is authorized using each one in the order it is specified. If one of them is not giving access, other one can give. So, every time a module denies a request, it goes to the next one in the chain, and as soon as a module approves the request, no more checks are done and the user is granted permissions.