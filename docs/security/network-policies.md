# Understanding Kubernetes NetworkPolicies

A `NetworkPolicy` controls and filters L3/L4 network traffic (Ingress & Egress) to and from Pods within a cluster using label selectors, namespace selectors, and CIDR blocks.

---

## 1. Core Concepts & Default Behavior

- **Default State (Non-Isolated):** By default, Kubernetes pods accept traffic from any source and send traffic to any destination.
- **Isolated State:** As soon as a pod is selected by a `NetworkPolicy` under `spec.podSelector`, it becomes **isolated** for the directions listed in `spec.policyTypes` (`Ingress`, `Egress`, or both).
- **Default Deny:** Once isolated, any traffic not explicitly permitted by a rule is dropped.
- **Requirement:** NetworkPolicies require a CNI plugin that supports policy enforcement (e.g., Calico, Cilium, Weave Net). Flannel does **not** support NetworkPolicies on its own.

---

## 2. The Three Selectors (`from` / `to`)

Traffic sources (`from` in Ingress) or destinations (`to` in Egress) are filtered using three building blocks:

| Selector | Scope / Target | Use Case |
| :--- | :--- | :--- |
| `podSelector` | Pods in the **same** namespace (or paired with `namespaceSelector`) | Target specific workloads (e.g., `role: frontend`) |
| `namespaceSelector` | All pods in a selected namespace | Target an entire environment (e.g., `dev`, `prod`) |
| `ipBlock` | External IP ranges (CIDR notation) | Traffic to/from on-prem servers, external APIs, etc. |

---

## 3. The AND vs. OR Rule (Crucial CKA Gotcha)

In YAML, array dashes (`-`) define list elements (**OR**), while key-value pairs at the same indentation define a single item (**AND**).

### A. The "AND" Condition (Combined in One List Item)
Traffic is allowed **only** if the pod matches `name: api-pod` **AND** resides in the namespace `kubernetes.io/metadata.name: dev`.

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            name: api-pod
        namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: dev
```

### B. The "OR" Condition (Separate List Items)
Traffic is allowed if the source is from the `dev` namespace **OR** comes from the external IP `192.168.5.10/32`.

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: dev
      - ipBlock:
          cidr: 192.168.5.10/32
```

> ⚠️ **Important Caveats:**
> - If `namespaceSelector` is omitted and only `podSelector` is supplied, Kubernetes only matches pods in the **same namespace** as the NetworkPolicy itself.
> - To target another namespace by name, always match the default label assigned by Kubernetes: `kubernetes.io/metadata.name: <namespace-name>`.
> - `ipBlock` cannot match cluster internal pod IPs reliably because pod IPs are ephemeral and dynamically assigned. Use `ipBlock` strictly for traffic entering or leaving the cluster.

---

## 4. Comprehensive Policy Manifest (Ingress + Egress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: development
spec:
  # Pods affected by this policy
  podSelector:
    matchLabels:
      role: db
  # Declare which directions are governed by this policy
  policyTypes:
    - Ingress
    - Egress

  # INGRESS RULES: Who can connect TO the db pod?
  ingress:
    - from:
        # Rule 1 (AND): Only 'api-pod' in the 'dev' namespace
        - podSelector:
            matchLabels:
              name: api-pod
          namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: dev
        # Rule 2 (OR): An external backup server IP
        - ipBlock:
            cidr: 192.168.5.10/32
      ports:
        - protocol: TCP
          port: 3306

  # EGRESS RULES: Where can the db pod connect OUT TO?
  egress:
    - to:
        - ipBlock:
            cidr: 192.168.5.10/32
      ports:
        - protocol: TCP
          port: 80
```

---

## 5. Deep-Dive Example: Analysis of `internal-policy`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - {}
  egress:
    - to:
        - podSelector:
            matchLabels:
              name: mysql
      ports:
        - protocol: TCP
          port: 3306
    - to:
        - podSelector:
            matchLabels:
              name: payroll
      ports:
        - protocol: TCP
          port: 8080
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

### Breakdown of Behavior:

1. **Target Pods (`spec.podSelector`):**
   - Applies to all pods in the namespace with label `name: internal`.

2. **Ingress Behavior (`ingress: - {}`):**
   - **Allow ALL Ingress.** An empty rule (`- {}`) under `ingress` acts as a wildcard, allowing incoming traffic from any pod, namespace, or IP on any port.

3. **Egress Behavior (Strictly Filtered):**
   - **Allow Egress to MySQL:** Can initiate outbound TCP connections to any pod in the same namespace labeled `name: mysql` on port `3306`.
   - **Allow Egress to Payroll:** Can initiate outbound TCP connections to any pod in the same namespace labeled `name: payroll` on port `8080`.
   - **Allow DNS Resolution:** 
     - Notice there is **no `to` field** under the 3rd egress item, only `ports: 53 (UDP & TCP)`.
     - When `to` is omitted, the rule allows traffic to **all destinations** (cluster DNS, CoreDNS, external DNS) on those specific ports.
     - *Why this is essential:* Without port 53 open on egress, the internal pod cannot resolve hostnames like `mysql.default.svc.cluster.local`.

---

## 6. Fast Imperative & Inspection Commands (CKA Exam)

> ⚠️ **Important Caveat:** There is **no imperative CLI command** like `kubectl create networkpolicy ...` that can build rules. You must write or patch YAML.

### Quick Template Generation
Generate a skeleton quickly during the exam using `kubectl`:

```bash
# Generate base skeleton (or copy from the official k8s documentation)
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress: []
EOF
```

### Inspection & Verification

```bash
# List NetworkPolicies
kubectl get networkpolicy -n <namespace>
kubectl get netpol -n <namespace>

# Inspect policy rules and selectors
kubectl describe netpol <policy-name> -n <namespace>

# Test network connectivity between pods
kubectl run test-curl --rm -it --image=busybox:1.28 --restart=Never -- nc -zvw 2 <target-pod-ip> 3306
kubectl run test-curl --rm -it --image=curlimages/curl --restart=Never -- curl -m 3 http://<target-pod-ip>:80
```