The **Kubernetes API Server (`kube-apiserver`)** is a standard RESTful HTTP server. Every operation executed via `kubectl`, client libraries, or internal controllers translates into an HTTP request directed at one of these top-level API paths.
## Root Endpoints
- `/metrics`
- `/healthz` (and `/livez`, `/readyz`)
- `/version`
- `/api`
- `/apis`
- `/logs`

These endpoints are divided into two primary categories:
1. **System & Health Endpoints** (Non-resource paths)
2. **Kubernetes Workload & Resource Endpoints** (`/api` vs `/apis`)

### A. The Core Group (`/api`)

The **Core Group** (sometimes called the Legacy Group) contains the original resources from early Kubernetes versions.

- **Group Name:** None (empty string `""`).
- **URL Pattern:** `/api/<version>/...`
- **Current Version:** `v1`
- **Common Resources Under `/api/v1`:**
    - `pods`
    - `services`
    - `namespaces`
    - `events`
    - `bindings`
    - `nodes`
    - `configmaps`
    - `secrets`
    - `persistentvolumes`, `persistentvolumeclaims`

> Why `apiVersion: v1` in YAML?
> When creating a Pod or Service, `apiVersion` has no group prefix because it lives directly under `/api/v1`

### B. Named API Groups (`/apis`)

As Kubernetes expanded, the API was modularized to allow distinct development lifecycles, versions, and extension capabilities (like Custom Resource Definitions). All new resources are organized under `/apis`.

- **Group Name:** Explicit (e.g., `apps`, `batch`, `networking.k8s.io`).
- **URL Pattern:** `/apis/<group>/<version>/...`
- **Common Named Groups:**

|API Group|Versions|Example Resources|YAML `apiVersion`|
|---|---|---|---|
|**apps**|`v1`|Deployments, ReplicaSets, StatefulSets, DaemonSets|`apiVersion: apps/v1`|
|**batch**|`v1`|Jobs, CronJobs|`apiVersion: batch/v1`|
|**networking.k8s.io**|`v1`|Ingress, NetworkPolicies|`apiVersion: networking.k8s.io/v1`|
|**storage.k8s.io**|`v1`|StorageClasses, VolumeAttachments|`apiVersion: storage.k8s.io/v1`|
|**autoscaling**|`v2`|HorizontalPodAutoscalers|`apiVersion: autoscaling/v2`|

> Why `apiVersion: apps/v1` in YAML?
> When defining a Deployment, the group name (`apps`) and version (`v1`) must both be included because its API route is `/apis/apps/v1/namespaces/<ns>/deployments`

-----

## 4. How REST Paths Map to Cluster Actions

Every `kubectl` request is translated into an explicit REST URI path:

|Action|Target REST Endpoint|
|---|---|
|`kubectl get pods -n default`|`GET /api/v1/namespaces/default/pods`|
|`kubectl get nodes`|`GET /api/v1/nodes` (cluster-scoped, no namespace)|
|`kubectl get deployments -n prod`|`GET /apis/apps/v1/namespaces/prod/deployments`|
|`kubectl get jobs -n dev`|`GET /apis/batch/v1/namespaces/dev/jobs`|

---

## 5. Helpful Commands to Inspect API Groups

### 1. Discover all resources and their API groups

```sh
kubectl api-resources
```
Look at the **`APIVERSION`** column:

- `v1` indicates the Core group (`/api/v1`).
- `<group>/<version>` (e.g., `apps/v1`) indicates a Named group under `/apis`.

### 2. List all available API versions

```sh
kubectl api-versions
```
This lists all active API groups and versions registered with the cluster.

### 3. Browse API Server Endpoints Directly

Using `kubectl proxy` opens an authenticated bridge to the API server:

```sh
# Start the local proxy
kubectl proxy --port=8001 &

# Query the root paths
curl http://localhost:8001/

# Query the core group
curl http://localhost:8001/api/v1

# Query named groups
curl http://localhost:8001/apis
curl http://localhost:8001/apis/apps/v1
```

> Note: `kube-proxy` and `kubectl proxy` are different