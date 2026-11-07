# Ingress vs. Gateway API: Same Rule, Two Ways

This page takes common routing rules and writes each one twice: once with **Ingress** (`networking.k8s.io/v1`) and once with **Gateway API** (`gateway.networking.k8s.io/v1`). The goal is to see where the two APIs differ in practice.

Read each example in this order:
1. The **goal**: what the rule should do.
2. The **Ingress** YAML: how you write it today.
3. The **Gateway API** YAML: how the same rule is written in the new API.
4. **What changed**: the important differences.

> **Note on Ingress examples:** The annotations used here (`nginx.ingress.kubernetes.io/...`) belong to the NGINX Ingress Controller. Other controllers use different annotation names, or none at all. That difference is the point of this page.

> **Setup used in the examples**
> - Namespace of the app: `apps`
> - Gateway namespace: `gateway-infra`
> - Gateway name: `prod-gateway`
> - Domain: `store.example.com`
> - Services: `product-service` (port 8080), `cart-service` (port 8080)

---

## 1. Basic Host and Path Routing

**Goal:** Requests to `store.example.com/products` go to `product-service`.

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-ingress
  namespace: apps
spec:
  ingressClassName: nginx
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8080
```

### Gateway API

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: store-route
  namespace: apps
spec:
  parentRefs:
    - name: prod-gateway
      namespace: gateway-infra
  hostnames:
    - store.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /products
      backendRefs:
        - name: product-service
          port: 8080
```

### What changed

- Ingress uses `ingressClassName` to pick the controller. Gateway API uses `parentRefs` to point at a specific **Gateway** object. The Gateway is owned by another team.
- `rules[].host` becomes `hostnames`. It is a list, so one route can serve several domains.
- `pathType: Prefix` becomes `matches[].path.type: PathPrefix`.
- `backend.service` becomes `backendRefs`, which is a list. That list is what makes traffic splitting possible later.

---

## 2. TLS (HTTPS) Configuration

**Goal:** Serve `store.example.com` over HTTPS using a certificate stored in a Secret called `store-tls`.

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-ingress
  namespace: apps
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - store.example.com
      secretName: store-tls
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8080
```

### Gateway API

TLS is configured on the **Gateway** listener, not on the route. The `prod-gateway` listener is written once by the cluster operator:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: gateway-infra
spec:
  gatewayClassName: eg-gateway-class
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: store.example.com
      tls:
        mode: Terminate
        certificateRefs:
          - name: store-tls
      allowedRoutes:
        namespaces:
          from: All
```

The route then only needs to attach to the `https` listener:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: store-route
  namespace: apps
spec:
  parentRefs:
    - name: prod-gateway
      namespace: gateway-infra
      sectionName: https
  hostnames:
    - store.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /products
      backendRefs:
        - name: product-service
          port: 8080
```

### What changed

- Ingress puts TLS in the **Ingress**, next to the routes. Gateway API puts TLS in the **Gateway**, so the cluster operator owns it and application developers cannot change it.
- Ingress uses `secretName`. Gateway API uses `certificateRefs`, which is a list.
- Gateway API makes `mode: Terminate` explicit. Ingress does this implicitly.
- `sectionName: https` tells the route which listener to use.

---

## 3. HTTP to HTTPS Redirect

**Goal:** Any request to `http://store.example.com` is redirected to `https://store.example.com`.

### Ingress

The redirect is an NGINX-only annotation:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-ingress
  namespace: apps
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - store.example.com
      secretName: store-tls
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8080
```

### Gateway API

The redirect is a **filter** on a route attached to the HTTP listener. It is standard YAML:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: store-http-redirect
  namespace: apps
spec:
  parentRefs:
    - name: prod-gateway
      namespace: gateway-infra
      sectionName: http
  hostnames:
    - store.example.com
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```

### What changed

- Ingress needs a controller-specific annotation. The same annotation does nothing on Traefik or Envoy.
- Gateway API uses the standard `RequestRedirect` filter. It works the same way on any conformant controller.
- The redirect is its own route, attached to the `http` listener. That keeps HTTP and HTTPS handling separate.

---

## 4. Path Rewrite

**Goal:** A request to `/shop/products/42` should reach the backend as `/products/42`. The `/shop` prefix is removed.

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-ingress
  namespace: apps
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /shop(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: product-service
                port:
                  number: 8080
```

The rewrite depends on a regex capture group (`$2`). The meaning of `ImplementationSpecific` also depends on the controller.

### Gateway API

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: store-route
  namespace: apps
spec:
  parentRefs:
    - name: prod-gateway
      namespace: gateway-infra
      sectionName: https
  hostnames:
    - store.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /shop
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: product-service
          port: 8080
```

### What changed

- Ingress needs regex mode, a capture group, and `rewrite-target` annotations. Regex syntax can differ between controllers.
- Gateway API uses `URLRewrite` with `ReplacePrefixMatch`. It replaces the matched prefix with a new value, with no regex needed.

---

## 5. Canary Release (Traffic Splitting by Weight)

**Goal:** Send 80% of traffic to `product-service-v1` and 20% to `product-service-v2`.

### Ingress

Ingress has no native split. You need a **second** Ingress that is marked as canary, and it has to point at the same host and path as the main Ingress:

```yaml
# Primary Ingress (normal Ingress, no annotations)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-primary
  namespace: apps
spec:
  ingressClassName: nginx
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product-service-v1
                port:
                  number: 8080
---
# Canary Ingress (takes 20% of the traffic that the primary would serve)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-canary
  namespace: apps
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  ingressClassName: nginx
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product-service-v2
                port:
                  number: 8080
```

### Gateway API

One route, two backends, two weights:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: store-route
  namespace: apps
spec:
  parentRefs:
    - name: prod-gateway
      namespace: gateway-infra
      sectionName: https
  hostnames:
    - store.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /products
      backendRefs:
        - name: product-service-v1
          port: 8080
          weight: 80
        - name: product-service-v2
          port: 8080
          weight: 20
```

### What changed

- With Ingress, the split is **spread across two objects** and depends on which one is the "main" one. You have to read both to know the real behavior.
- With Gateway API, the whole split is in one place. The weights are visible in one file.
- The Ingress canary annotations only work on NGINX. Gateway API weights work on any conformant controller.

---

## 6. Header-Based Routing (Beta Testers)

**Goal:** Requests with the header `X-Beta: true` go to `product-service-beta`. All other requests go to `product-service`.

### Ingress

```yaml
# Canary Ingress, which only matches requests with the header
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-beta
  namespace: apps
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Beta"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product-service-beta
                port:
                  number: 8080
```

### Gateway API

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: store-route
  namespace: apps
spec:
  parentRefs:
    - name: prod-gateway
      namespace: gateway-infra
      sectionName: https
  hostnames:
    - store.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /products
          headers:
            - name: X-Beta
              value: "true"
      backendRefs:
        - name: product-service-beta
          port: 8080
    - matches:
        - path:
            type: PathPrefix
            value: /products
      backendRefs:
        - name: product-service
          port: 8080
```

### What changed

- Ingress needs a second, canary-marked Ingress with three annotations.
- Gateway API uses `headers` inside `matches`. The rule order matters: the more specific rule (with the header) comes first. The second rule catches everything else.
- In Gateway API, `path` and `headers` inside the same match item are **AND**ed. Both must match.

---

## 7. Cross-Namespace Routing

**Goal:** The `store` route in `apps` should send traffic to a Service called `auth-service` in the namespace `security`.

### Ingress

Ingress can only point to a Service in **its own namespace**. There is no standard way to reference another namespace. The usual workaround is an `ExternalName` Service that points to the other namespace, and that is a hack, not a proper solution.

### Gateway API

Gateway API allows this, but the owner of the target namespace has to **approve** it with a `ReferenceGrant`.

The route in `apps`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: auth-route
  namespace: apps
spec:
  parentRefs:
    - name: prod-gateway
      namespace: gateway-infra
      sectionName: https
  hostnames:
    - store.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /auth
      backendRefs:
        - name: auth-service
          namespace: security
          port: 8080
```

The grant in `security`, which allows routes from `apps` to use the Service:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-apps-to-auth
  namespace: security
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: apps
  to:
    - group: ""
      kind: Service
      name: auth-service
```

### What changed

- Ingress has no clean cross-namespace model.
- Gateway API makes the permission explicit. The `security` team decides who can use `auth-service`, and nobody can reach into their namespace without that grant.

> **Note:** `ReferenceGrant` is served at `v1beta1` in many releases. Check which version your cluster exposes with `kubectl api-resources | grep -i referencegrant`.

---

## 8. TCP Routing (for Example, a Database)

**Goal:** Expose a PostgreSQL Service (`db-service`, port 5432) through the gateway on port 5432.

### Ingress

Ingress cannot route TCP. With NGINX, you have to edit a global **ConfigMap** called `tcp-services` in the controller's namespace. That ConfigMap is shared by the whole cluster, and it is not part of the Ingress object at all:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  "5432": "apps/db-service:5432"
```

### Gateway API

A `TCPRoute` is a normal, namespaced object, but it needs a matching **listener** on the Gateway:

```yaml
# Added to the prod-gateway listeners
- name: postgres
  protocol: TCP
  port: 5432
  allowedRoutes:
    kinds:
      - kind: TCPRoute
    namespaces:
      from: Same
```

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: db-route
  namespace: apps
spec:
  parentRefs:
    - name: prod-gateway
      namespace: gateway-infra
      sectionName: postgres
  rules:
    - backendRefs:
        - name: db-service
          port: 5432
```

### What changed

- Ingress has no TCP support. The NGINX workaround is global and lives outside the Ingress object.
- Gateway API defines a TCP listener and a `TCPRoute`. The route belongs to the app team, and the listener belongs to the operator.
- `TCPRoute` is still at `v1alpha2` in many releases, so check your cluster's installed version.

---

## 9. CORS Headers

**Goal:** Allow browsers from `https://app.example.com` to call the API.

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-ingress
  namespace: apps
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
spec:
  ingressClassName: nginx
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8080
```

### Gateway API

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: store-route
  namespace: apps
spec:
  parentRefs:
    - name: prod-gateway
      namespace: gateway-infra
      sectionName: https
  hostnames:
    - store.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /products
      filters:
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            set:
              - name: Access-Control-Allow-Origin
                value: https://app.example.com
      backendRefs:
        - name: product-service
          port: 8080
```

### What changed

- Ingress has a built-in CORS annotation set that only NGINX reads.
- Gateway API sets the header directly with `ResponseHeaderModifier`. It is a plain header change, so it works the same on every controller.
- Gateway API does not have a full CORS feature in the core spec. For the complete set of CORS headers (preflight, methods, credentials), you may need several `set` entries. Check your controller's support for `OPTIONS` handling.

---

## 10. Summary Table

| Rule | Ingress | Gateway API |
| :--- | :--- | :--- |
| Host and path | `rules[].host` and `paths[]` | `hostnames` and `matches[].path` |
| TLS | `spec.tls` with `secretName` | Listener `tls.certificateRefs` on the Gateway |
| HTTP to HTTPS redirect | `ssl-redirect` annotation (NGINX only) | `RequestRedirect` filter |
| Path rewrite | `rewrite-target` annotation and regex | `URLRewrite` filter |
| Traffic split | Second Ingress with `canary-weight` (NGINX only) | Multiple `backendRefs` with `weight` |
| Header routing | `canary-by-header` annotation (NGINX only) | `headers` in `matches` |
| Cross-namespace Service | Not supported cleanly | `ReferenceGrant` |
| TCP and UDP | Not supported (global ConfigMap workaround) | `TCPRoute`, `UDPRoute` |
| CORS and header changes | Annotations (NGINX only) | `ResponseHeaderModifier` filter |
| Who owns what | One object for everyone | Separate objects for infra, operator, and app team |

---

## Key Takeaways

- **Annotations are the main difference.** Most Ingress features beyond host and path are controller-specific annotations. Gateway API moves them into standard fields.
- **Gateway API spreads ownership.** TLS and listeners belong to the operator. Routes belong to the app team. Cross-namespace access needs an explicit grant.
- **Ingress is simpler for small setups.** If you only need host and path routing with one team, a single Ingress is still fine.
