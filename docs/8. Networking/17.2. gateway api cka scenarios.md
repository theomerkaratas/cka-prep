# Gateway API in the CKA Exam: Common Scenarios and Simple Troubleshooting

The **Gateway API** (`gateway.networking.k8s.io/v1`) graduated to General Availability (GA) and is increasingly tested in Kubernetes administrator certification environments (CKA and CKAD).

Exam tasks typically focus on:
1. **Inspecting cluster GatewayClasses and active Gateways.**
2. **Creating an `HTTPRoute` to connect a backend Service to an existing Gateway.**
3. **Configuring Canary traffic splitting (`weight`).**
4. **Troubleshooting route attachment failures and status conditions.**

Here are the key scenarios and simple, easy-to-remember commands.

---

## Quick Reference: Resource Hierarchy

```text
GatewayClass (Cluster-wide)  ──►  Gateway (Infra / Operator)  ──►  HTTPRoute (Developer / App)
```

| Resource | Short Name | API Group | Scope |
| :--- | :--- | :--- | :--- |
| `GatewayClass` | `gc` | `gateway.networking.k8s.io/v1` | Cluster |
| `Gateway` | `gtw` | `gateway.networking.k8s.io/v1` | Namespaced |
| `HTTPRoute` | `httproute` | `gateway.networking.k8s.io/v1` | Namespaced |
| `ReferenceGrant` | `refgrant` | `gateway.networking.k8s.io/v1beta1` | Namespaced |

---

## Scenario 1: Inspecting GatewayClass and Gateway Status

### Exam Task
> *"Identify the available GatewayClass and verify the IP address assigned to the `main-gateway` in namespace `infra`."*

### Step 1: List GatewayClasses
```bash
kubectl get gatewayclass
# or shortname:
kubectl get gc
```
*Example output:*
```text
NAME      CONTROLLER                                      ACCEPTED   AGE
default   gateway.envoyproxy.io/gatewayclass-controller   True       10d
```

### Step 2: Inspect the Gateway
```bash
kubectl get gateway -n infra
# or shortname:
kubectl get gtw -n infra
```
*Example output:*
```text
NAME           CLASS     ADDRESS        PROGRAMMED   AGE
main-gateway   default   192.168.1.50   True         5d
```
* **Class:** `default`
* **Assigned IP:** `192.168.1.50`
* **PROGRAMMED `True`:** Indicates the Gateway controller successfully created the network load balancer.

---

## Scenario 2: Create an `HTTPRoute` for a Backend Service

### Exam Task
> *"Create an HTTPRoute named `web-route` in namespace `prod` that binds to the Gateway `main-gateway` in namespace `infra`. Route requests matching host `example.com` and path `/web` to service `web-service:80`."*

### Step 1: Write the `HTTPRoute` Manifest
Create `web-route.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: prod
spec:
  parentRefs:
    - name: main-gateway
      namespace: infra
      sectionName: http         # Optional: listener name on the Gateway
  hostnames:
    - example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /web
      backendRefs:
        - name: web-service
          port: 80
```

### Step 2: Apply and verify attachment
```bash
kubectl apply -f web-route.yaml
kubectl get httproute -n prod
```

Check if the route attached to the Gateway:
```bash
kubectl describe httproute web-route -n prod
```
Look for:
```text
Status:
  Parents:
    Conditions:
      Type:    Accepted
      Status:  True
      Reason:  Accepted
```

---

## Scenario 3: Canary Traffic Splitting using Weights

### Exam Task
> *"Configure the existing HTTPRoute `api-route` in namespace `apps` to split traffic: 80% to `api-v1:8080` and 20% to `api-v2:8080`."*

### Step 1: Edit the HTTPRoute
```bash
kubectl edit httproute api-route -n apps
```

### Step 2: Configure `backendRefs` with `weight`
Update the `rules.backendRefs` list:

```yaml
spec:
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-v1
          port: 8080
          weight: 80           # 80% of requests
        - name: api-v2
          port: 8080
          weight: 20           # 20% of requests
```
Save and exit (`:wq`). Traffic is now dynamically split by the gateway proxy according to the specified weights.

---

## Scenario 4: Troubleshooting Unattached Routes (`Accepted: False`)

### Exam Task
> *"An HTTPRoute `test-route` was deployed in namespace `dev`, but traffic sent to the Gateway is not reaching `test-service`. Troubleshoot and fix."*

### Step 1: Check HTTPRoute Status Conditions
```bash
kubectl describe httproute test-route -n dev
```

Look at the **Conditions** section under `Status.Parents`:

* **Error 1: `Reason: NotAllowedByListeners`**  
  The Gateway does not permit routes from the `dev` namespace!  
  *Fix:* Inspect the Gateway (`kubectl describe gtw main-gateway -n infra`). Check `allowedRoutes.namespaces.from`. If it says `Same`, routes from `dev` are blocked. Update `from` to `All` or add a namespace selector.
* **Error 2: `Reason: BackendNotFound`**  
  The backend Service name or port in `backendRefs` does not exist in namespace `dev`.  
  *Fix:* Verify `kubectl get svc -n dev` and match the Service name and port.
* **Error 3: `Reason: SectionNameNotFound`**  
  The `sectionName` in `parentRefs` does not match any listener name defined on the Gateway.

---

## Summary: Simple Command Workflow for CKA

```bash
# 1. View GatewayClasses and active Gateways:
kubectl get gc
kubectl get gtw -A

# 2. View HTTPRoutes across all namespaces:
kubectl get httproute -A

# 3. Check attachment status and troubleshooting errors:
kubectl describe httproute <name> -n <ns>

# 4. Check Gateway controller logs:
kubectl logs -n <controller-namespace> -l app=<controller-name>
```

