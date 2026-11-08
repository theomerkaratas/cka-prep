# Kubernetes ServiceAccount: Comprehensive Guide

In Kubernetes, authentication is divided into two main categories: humans (**Users**) and internal applications/automated processes (**ServiceAccounts**).

When an application running inside a Pod (such as Prometheus, an Ingress Controller, or a CI/CD runner) needs to communicate with the Kubernetes API Server, it authenticates using a **ServiceAccount**.

---

## 1. Fundamentals and Namespace Scope

- **Namespaced Resource:** Every ServiceAccount is created within a specific namespace.
- **The `default` Account:** When a cluster is provisioned or a new namespace is created, Kubernetes automatically generates a ServiceAccount named **`default`** inside that namespace.
- **Cross-Namespace Actions:** Although a ServiceAccount lives in a specific namespace, its permissions are not restricted to that namespace. By binding it to a **RoleBinding** in another namespace or to a **ClusterRoleBinding** across the cluster, a ServiceAccount can perform operations in other namespaces or across the entire cluster.

List existing ServiceAccounts:
```bash
kubectl get serviceaccount
# or shorthand:
kubectl get sa
```

---

## 2. Pod and ServiceAccount Relationship: How Tokens Work

When a Pod is created without specifying a custom account, Kubernetes automatically associates it with the **`default`** ServiceAccount of that namespace.

```bash
kubectl describe pod my-kubernetes-dashboard
```

The output highlights two key configuration items:
1. `Service Account: default`
2. A **Projected Volume** mounted under `Volumes`.

```text
Containers:
  web-dashboard:
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount
Volumes:
  kube-api-access-rtvkd:
    Type:                    Projected
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
```

### Projected Volumes and Injected Files
Kubernetes dynamically mounts a projected volume into the container filesystem. Inspecting that directory reveals three critical files:

```bash
kubectl exec -it my-kubernetes-dashboard -- ls /var/run/secrets/kubernetes.io/serviceaccount
# Output: ca.crt  namespace  token
```

- **`token`:** A short-lived, signed JSON Web Token (JWT) proving the Pod's identity to the API Server.
- **`ca.crt`:** The certificate authority bundle used to verify the API Server's TLS identity during HTTPS handshakes.
- **`namespace`:** A plain-text file containing the name of the namespace the Pod runs in.

### Token Architecture (Bound ServiceAccount Tokens)
In modern Kubernetes versions:
- **Time-bound:** Tokens carry an expiration timestamp (typically 1 hour by default).
- **Automated rotation:** The kubelet automatically requests and refreshes the token before it expires.
- **Pod-bound:** The token is tied directly to the lifetime of the Pod. If the Pod is deleted, the token becomes invalid.
- **Standard mechanism:** Cryptographic JWT bearer tokens are the standard and universal authentication method for ServiceAccounts.

---

## 3. Creating Custom ServiceAccounts and Attaching to Pods

Because the `default` ServiceAccount has minimal permissions, custom ServiceAccounts are created to satisfy specific RBAC requirements.

### A. Creating a ServiceAccount

**Imperative (CLI):**
```bash
kubectl create serviceaccount build-robot -n development
```

**Declarative (YAML):**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
  namespace: development
```

### B. Attaching to a Pod
Specify the ServiceAccount name in the Pod specification using **`spec.serviceAccountName`**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-k8s-dashboard
  namespace: development
spec:
  serviceAccountName: build-robot
  containers:
    - name: web-dashboard
      image: nginx
```

---

## 4. Security Hardening: Disabling Token Automounting (`automountServiceAccountToken`)

Many workloads (such as standard web applications, microservices, or databases) never make calls to the Kubernetes API Server. Mounting credentials into these containers introduces unnecessary exposure; if the container is compromised, the attacker can leverage the mounted token to query the cluster.

To prevent Kubernetes from injecting the token, configure **`automountServiceAccountToken: false`**.

### Option 1: At the ServiceAccount Level
Disables automounting for all Pods that reference this ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-token-sa
  namespace: development
automountServiceAccountToken: false
```

### Option 2: At the Pod Level
Overrides the ServiceAccount configuration and explicitly blocks the volume mount for that specific Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: development
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
  containers:
    - name: app
      image: nginx
```

---

## 5. External Usage & Manual Token Generation

In scenarios where external systems (such as external CI/CD pipelines, monitoring agents, or administrative dashboards) must authenticate against the cluster, tokens can be generated on demand using `kubectl create token`:

```bash
# Generate a short-lived token for the build-robot account:
kubectl create token build-robot -n development

# Customize the expiration duration:
kubectl create token build-robot -n development --duration=24h
```

Key characteristics:
- The token is output directly to stdout as a JWT string and is **not stored as a Secret object** in etcd.
- External systems can consume it as a bearer token in standard HTTPS API requests:

```bash
curl -k https://<controlplane-ip>:6443/api/v1/namespaces/development/pods \
  -H "Authorization: Bearer <GENERATED_TOKEN>"
```

---

## 6. Granular Control: Manual Token Injection via Projected Volumes

Disabling automatic mounting (`automountServiceAccountToken: false`) does not mean the pod can never receive a token. You can choose to **disable automated injection** while still **manually mounting a token** using an explicit `projected` volume.

### Why Use Manual Projection?
- **Least Privilege Path:** Mount only the `token` file instead of everything (`ca.crt`, `namespace`).
- **Fine-Grained Settings:** Explicitly define the target path, mount mode (`readOnly: true`), token expiration (`expirationSeconds`), and audience (`audience`).
- **Strict Compliance:** Adhere to security standards requiring explicit volume definitions rather than magic defaults.

### Declarative Example (Deployment / Pod Spec):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-dashboard
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      name: web-dashboard
  template:
    metadata:
      labels:
        name: web-dashboard
    spec:
      serviceAccountName: dashboard-sa
      automountServiceAccountToken: false  # 1. Disable default automount

      containers:
        - name: web-dashboard
          image: nginx
          volumeMounts:
            # 2. Mount the projected volume explicitly and read-only
            - name: token-vol
              mountPath: /var/run/secrets/kubernetes.io/serviceaccount/
              readOnly: true

      volumes:
        # 3. Define the projected volume with serviceAccountToken source
        - name: token-vol
          projected:
            sources:
              - serviceAccountToken:
                  path: token              # Injected as .../serviceaccount/token
                  expirationSeconds: 3600  # Optional: custom TTL (e.g. 1 hour)
```

------
## Summary Lifecycle

1. **Identity:** Provides machine identity to Pods and automated tooling interacting with the Kubernetes API.
2. **Assignment:** Attached to workloads using `spec.serviceAccountName`.
3. **Mounting:** Injected by the kubelet as a Projected Volume at `/var/run/secrets/kubernetes.io/serviceaccount` containing `token`, `ca.crt`, and `namespace`.
4. **Authorization:** Governed via **RoleBinding** (namespace-scoped) or **ClusterRoleBinding** (cluster-scoped).
5. **Hardening:** When API access is unnecessary, disable token injection using `automountServiceAccountToken: false`.