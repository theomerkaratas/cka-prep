# Security Primitives

Kubernetes security follows a **defense-in-depth** model. Securing a cluster involves safeguarding multiple layers: the underlying host infrastructure, the control plane API, inter-component communication, workload execution, and network traffic.

```text
               +---------------------------------------+
               | 1. Host & Infrastructure Security     |
               | (SSH keys, OS patches, root password) |
               +-------------------+-------------------+
                                   |
                                   v
               +---------------------------------------+
               | 2. API Server Access Control          |
               | [AuthN] -> [AuthZ] -> [Admission]     |
               +-------------------+-------------------+
                                   |
                                   v
               +---------------------------------------+
               | 3. Cluster Transport Security (TLS)   |
               | (mTLS across all internal components) |
               +-------------------+-------------------+
                                   |
                                   v
               +---------------------------------------+
               | 4. Workload & Network Security        |
               | (SecurityContext & NetworkPolicies)   |
               +---------------------------------------+
```

---

## 1. Host & Infrastructure Security

- **Host Access:** Worker and control plane nodes must be accessible only via SSH key-based authentication with root password logins disabled.
- **Firewall Rules:** Block all unnecessary inbound ports. Only expose required ports (e.g. `6443` for API server, `10250` for Kubelet, `30000-32767` for NodePort services).

---

## 2. API Server Access Control

All management operations pass through the `kube-apiserver`. Requests must clear three sequential security gates:

### A. Authentication ("Who are you?")
Determines the identity of the requester:
- **Human Users:** TLS Client Certificates (X.509), OIDC (OAuth2/JWT tokens), or Authenticating Proxy.
- **Workloads / Pods:** `ServiceAccount` tokens injected automatically by the `kubelet`.

### B. Authorization ("What can you do?")
Determines what actions the authenticated identity can perform:
- **RBAC (Role-Based Access Control):** Standard declarative permissions via `Role`, `ClusterRole`, `RoleBinding`, and `ClusterRoleBinding`.
- **Node Authorization:** Restricts `kubelet` to read only its own node's pods, secrets, and configmaps.
- **Webhook Authorization:** Calls out to external authorization services (e.g. OPA Gatekeeper).

### C. Admission Control ("Can this request be admitted?")
Validates and mutates requests before persisting to `etcd`:
- **Built-in Plugins:** `NamespaceLifecycle`, `LimitRanger`, `ResourceQuota`, `NodeRestriction`.
- **Dynamic Webhooks:** `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook`.

---

## 3. Cluster Transport Security (TLS Encryption)

- All network communication within the cluster is encrypted using **mutual TLS (mTLS)**.
- Every component (`kube-apiserver`, `etcd`, `kube-controller-manager`, `kube-scheduler`, `kubelet`, `kube-proxy`) holds its own certificate-key pair and validates other components against the cluster Root CA (`ca.crt`).
- **`etcd` Isolation:** Only the `kube-apiserver` communicates directly with `etcd`. All other components and users access cluster state strictly through the API server.

---

## 4. Workload and Network Security

### Workload Hardening (`securityContext`)
- **Run as Non-Root:** Enforce `runAsNonRoot: true` to prevent container breakout exploits as UID 0.
- **Drop Capabilities:** Restrict Linux kernel capabilities using `capabilities: { drop: ["ALL"] }`.
- **Read-Only Root Filesystem:** Protect container filesystems against runtime tampering.

### Network Isolation (`NetworkPolicy`)
- **Default Behavior:** By default, Kubernetes networking is completely flat—any pod can communicate with any other pod across any namespace.
- **Enforcing Boundaries:** Use `NetworkPolicy` objects to isolate sensitive databases and backend services, allowing traffic only from authorized frontend tiers.
 
