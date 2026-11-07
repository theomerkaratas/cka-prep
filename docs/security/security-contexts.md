# Kubernetes SecurityContext Guide

The **`securityContext`** field allows you to define privilege and access control settings for Pods and containers directly in your Kubernetes manifests. It acts as the declarative Kubernetes interface for underlying Linux security mechanisms (such as UID/GID assignment, file system permissions, and Linux capabilities).

The most important architectural consideration is understanding whether a setting applies at the **Pod level** or the **Container level**.

---

## 1. Two-Tier Configuration: Pod-Level vs. Container-Level

In Kubernetes manifests, `securityContext` can be defined in two locations:

```text
spec:
  securityContext:      <-- 1. POD Level (Inherited by all containers)
    runAsUser: 1000

  containers:
    - name: web
      securityContext:  <-- 2. CONTAINER Level (Applies only to this container)
        runAsUser: 2000
```

### Inheritance and Override Rules:
1. **Inheritance:** When defined at the **Pod level** (`spec.securityContext`), the configuration applies to **all containers** within that Pod automatically.
2. **Override (Precedence):** If the same parameter is configured at both levels, the **Container-level setting overrides** the Pod-level setting.
   - In the example above, the `web` container executes with UID `2000`, overriding the Pod default of `1000`.

---

## 2. Critical Caveat: Capabilities are Container-Level Only!

Pay close attention to this standard Kubernetes limitation:

> **Important Rule:** Linux **`capabilities`** (and the **`privileged`** flag) are **only supported at the Container level**, never at the Pod level.

- Fields like `runAsUser`, `runAsGroup`, and `runAsNonRoot` can be placed at either the Pod or Container level.
- Attempting to place `capabilities` under `spec.securityContext` (Pod level) causes a schema validation failure (`unknown field "capabilities"`) and rejects the manifest.

---

## 3. Practical Examples

### Example A: Pod-Level Configuration
Use this when you want all containers in a Pod to execute under UID `1000` and GID `3000`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  securityContext: # Pod level
    runAsUser: 1000
    runAsGroup: 3000
  containers:
    - name: app
      image: ubuntu
      command: ["sleep", "3600"]
    - name: sidecar
      image: ubuntu
      command: ["sleep", "3600"]
```
*Both `app` and `sidecar` inherit and run with UID 1000 / GID 3000.*

---

### Example B: Container-Level Configuration & Override
Define a baseline Pod-level user, override it for a specific container, and grant dedicated Linux capabilities:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsUser: 1000 # Default for containers in this Pod

  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext: # Container level
        runAsUser: 2000 # Overrides Pod-level runAsUser
        capabilities:   # Must be defined at the container level
          add: ["NET_ADMIN", "MAC_ADMIN"]
          drop: ["ALL"]
```

---

## 4. Parameter Scope Reference Table

| Setting | Supported Scope | Description |
|---|:---:|---|
| **`runAsUser`** | Pod & Container | The Linux User ID (UID) used to run the container entrypoint process. |
| **`runAsGroup`** | Pod & Container | The Linux Primary Group ID (GID) used for process execution. |
| **`runAsNonRoot`** | Pod & Container | Validates that the container image does not run as root (UID 0). Fails startup if UID 0 is detected. |
| **`fsGroup`** | **Pod Only** | Supplementary group ID applied to volume mounts and filesystem ownership. |
| **`capabilities`** | **Container Only** | Linux capabilities to grant (`add`) or remove (`drop`). |
| **`privileged`** | **Container Only** | Grants full host-level root access and device access to the container. |
| **`readOnlyRootFilesystem`** | Pod & Container | Mounts the container root filesystem as read-only. |

---

## 5. Verification Commands

To verify that the security context settings took effect inside the running container:

```bash
# Check the username of the running process:
kubectl exec ubuntu-sleeper -- whoami
# Example output (if runAsUser is non-root or unmapped):
# 1000 (or username if defined in /etc/passwd)

# Check UID, GID, and assigned groups inside the container:
kubectl exec -it web-pod -- id
# Example output:
# uid=2000 gid=0(root) groups=0(root)

# Check granted Linux capabilities inside the container:
kubectl exec -it web-pod -- capsh --print
```