Here are your notes restructured into the clean, CKA-optimized study format:

---

# Understanding Custom Resource Definitions (CRDs) & Custom Controllers

A **CustomResourceDefinition (CRD)** extends the Kubernetes API by registering new declarative resource types (Kinds) with the API Server and storing their data in `etcd`, while a **Custom Controller** contains the operational reconciliation loop that acts upon those resources.

---

## 1. Core Architecture & Mental Model

- **CRD (The Data Structure):** Extends the API. It tells the `kube-apiserver` what the resource looks like, validates the manifest schema, and stores instances into `etcd`.
- **Custom Controller (The Brain):** Without a controller, a custom resource is just **passive data** in `etcd`. The controller continuously watches the API server and actively reconciles current state with desired state (Operator pattern).
- **Naming Convention Requirement:** The CRD `metadata.name` **must** strictly follow `<plural>.<group>` (e.g., `postgresqls.db.example.com`).

---

## 2. Anatomy of a CustomResourceDefinition (CRD)

### Declarative Manifest (YAML)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # MUST follow: <names.plural>.<group>
  name: postgresqls.db.example.com
spec:
  # Scope: Namespaced or Cluster
  scope: Namespaced
  # Custom API Group
  group: db.example.com
  # Resource naming aliases
  names:
    plural: postgresqls
    singular: postgresql
    kind: Postgresql
    shortNames:
      - pg
  # API Version Lifecycle management
  versions:
    - name: v1
      # Enables/disables this version in the REST API
      served: true
      # Dictates which version is persisted into etcd
      storage: true
      # OpenAPI v3 schema validation
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                storageGb:
                  type: integer
                  minimum: 10
                  maximum: 500
                version:
                  type: string
```

> ⚠️ **Important Caveats (Exam Traps):**
> - **API Version:** Notice the exact group is `apiextensions.k8s.io/v1` (with an **`s`** in `apiextensions`).
> - **Naming Match:** If `metadata.name` does not match `<names.plural>.<spec.group>`, the API server will reject the CRD with a validation error.
> - **Field Typo Trap:** The field is `scope: Namespaced` (or `Cluster`), **not** `scoped`.

---

## 3. Multiple Versions & The `storage` Flag

A single CRD can expose multiple versions as it evolves through its lifecycle (e.g., `v1alpha1`, `v1beta1`, `v1`).

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: flighttickets.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: flighttickets
    singular: flightticket
    kind: Flightticket
    shortNames:
      - ft
  versions:
    # Deprecated or transitional version
    - name: v1beta1
      served: true
      storage: false # Cannot store as v1beta1 in etcd
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                from:
                  type: string
                to:
                  type: string
                number:
                  type: integer
                  minimum: 1
                  maximum: 10
    # Current active storage version
    - name: v1
      served: true
      storage: true  # Exactly ONE version can be storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                from:
                  type: string
                to:
                  type: string
                number:
                  type: string
```

### Version Rule Summary:
- **`served: true/false`**: Determines whether clients (`kubectl`, API requests) can query/submit this version.
- **`storage: true/false`**: **Only one version** in the entire CRD can have `storage: true`. This version is used by the API server when writing the object into `etcd`.

---

## 4. Creating and Consuming the Custom Resource (CR)

Once the CRD is applied, the custom endpoint is immediately available.

### Instance Manifest (YAML)

```yaml
# Constructed as: <spec.group>/<version.name>
apiVersion: db.example.com/v1
kind: Postgresql
metadata:
  name: my-prod-db
  namespace: default
spec:
  storageGb: 50
  version: "15"
```

---

## 5. Inspection & Verification Commands

```bash
# Check if the CRD is registered in the cluster
kubectl get crd
kubectl get crd postgresqls.db.example.com

# Describe CRD details, validation schema, and versions
kubectl describe crd postgresqls.db.example.com

# Query your custom resources using plural, singular, or shortNames
kubectl get postgresqls
kubectl get postgresql
kubectl get pg -n default

# Inspect a specific custom resource instance
kubectl describe pg my-prod-db

# Check API resources to discover custom groups and short names
kubectl api-resources --api-group=db.example.com
```