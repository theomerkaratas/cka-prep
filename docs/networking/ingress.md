# Kubernetes Ingress: Architecture, Ingress Controllers, and Networking Mechanics

In Kubernetes, exposing applications to external clients using only standard **Services** (`NodePort` or `LoadBalancer`) introduces significant operational, architectural, and financial limitations:

1. **`NodePort` limitations:** Forces clients to remember non-standard high ports (`30000-32767`), exposes raw node IPs, and lacks SSL/TLS termination and Layer 7 (HTTP/HTTPS) routing capabilities.
2. **`LoadBalancer` limitations:** Every single Service of type `LoadBalancer` provisions an independent, paid cloud load balancer (e.g., AWS NLB/ALB, GCP Cloud Load Balancer). Exposing 20 microservices results in 20 public IP addresses, 20 cloud load balancers, and substantial cloud costs.

**Ingress** solves this problem by acting as an intelligent **Layer 7 Application Gateway (Reverse Proxy / HTTP Router)** for the entire cluster. It provides a single entry point (single IP address), routing external HTTP/HTTPS traffic to multiple internal Services based on **hostnames** and **request paths**, while handling **SSL/TLS termination**.

---

## 1. Core Concepts: Ingress Controller vs. Ingress Resource

To understand Ingress in Kubernetes, you must understand that Ingress is split into two completely separate parts:

```text
┌────────────────────────────────────────────────────────┐
│                   Ingress Controller                   │
│        (The Data Plane: NGINX, Traefik, HAProxy)       │
│  - Actual running Pod / Deployment                     │
│  - Listens on ports 80/443 for external traffic        │
│  - Auto-configures internal reverse proxy rules        │
└───────────────────────────▲────────────────────────────┘
                            │ Reads & Synchronizes
                            ▼
┌────────────────────────────────────────────────────────┐
│                    Ingress Resource                    │
│            (The Control Plane: YAML Object)            │
│  - Native Kubernetes API object (kind: Ingress)        │
│  - Defines routing rules (hosts, paths, backends)      │
│  - Defines TLS certificates and secrets                │
└────────────────────────────────────────────────────────┘
```

### 1. Ingress Controller (The Engine)
* Kubernetes **does NOT include a built-in Ingress Controller by default**. Creating an Ingress YAML file in a bare cluster will do absolutely nothing unless an Ingress Controller is installed.
* An Ingress Controller is a specialized reverse proxy deployed as a Kubernetes Deployment or DaemonSet (e.g., `ingress-nginx`, Traefik, Envoy, HAProxy).
* It continuously monitors the Kubernetes API for `Ingress` objects, dynamically generates its proxy configuration (e.g., `nginx.conf`), and reloads itself without dropping connections.

### 2. Ingress Resource (The Rules)
* A standard Kubernetes object (`apiVersion: networking.k8s.io/v1`, `kind: Ingress`) created and managed by administrators and developers.
* Defines the routing table: *"If traffic comes to `shop.example.com/checkout`, forward it to Service `checkout-service:80`."*

### 3. IngressClass
* Clusters can run multiple Ingress Controllers simultaneously (e.g., one internal controller for company intranets, one external controller for public web traffic).
* An **`IngressClass`** acts as the link between an Ingress Resource and a specific Ingress Controller. The Ingress YAML specifies `spec.ingressClassName: nginx` to declare which controller should process its rules.

---

## 2. Ingress vs. Service: What Is the Difference?

| Attribute | Service (`ClusterIP`, `NodePort`, `LoadBalancer`) | Ingress |
| :--- | :--- | :--- |
| **OSI Layer** | **Layer 4 (Transport Layer)**: Routes TCP/UDP packets by IP and Port. | **Layer 7 (Application Layer)**: Inspects HTTP headers, hostnames, paths, cookies, and TLS SNI. |
| **Traffic Handling** | Implemented in Linux kernel via `kube-proxy` (`iptables` / IPVS). | Processed in user-space by an actual reverse proxy process (e.g., NGINX daemon). |
| **SSL/TLS Termination** | No native support; must be handled by application Pods or external LBs. | Supported natively via Kubernetes `Secret` objects. |
| **Routing Mechanism** | Round-robin forwarding to matching Pod endpoints. | Content-based routing: Host-based (`app.com` vs `api.com`) or Path-based (`/users` vs `/orders`). |
| **Cost & Public IPs** | 1 cloud Load Balancer per Service. | 1 cloud Load Balancer for **unlimited** internal Services. |

---

## 3. The End-to-End Traffic Flow: Pod, Service, and Ingress

When deploying a production web application in Kubernetes, all three components work together in a strict pipeline:

```text
User / Client (Browser)
      │
      │ 1. HTTPS Request: https://shop.example.com/api
      ▼
┌──────────────────────────────────────────────────────────────────┐
│ Cloud Load Balancer / External Firewall                          │
│ IP: 203.0.113.50 (Public IP)                                     │
└─────────────────────────────────┬────────────────────────────────┘
                                  │ 2. Routes to Ingress NodePort
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│ Ingress Controller Service (Type: LoadBalancer / NodePort)       │
└─────────────────────────────────┬────────────────────────────────┘
                                  │ 3. Forwards to Ingress Pod
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│ Ingress Controller Pod (NGINX / HAProxy)                         │
│  - Terminates TLS using Secret "shop-tls"                        │
│  - Evaluates Ingress rules: host "shop.example.com", path "/api" │
│  - Selects Backend Service: "api-service:8080"                   │
└─────────────────────────────────┬────────────────────────────────┘
                                  │ 4. Direct forwarding via Endpoints
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│ Backend Pods (Running Application: Spring, Node, Python, etc.)   │
│ Pod IP: 10.244.1.4:8080, 10.244.2.9:8080                         │
└──────────────────────────────────────────────────────────────────┘
```

> [!NOTE]
> ### How Ingress Bypasses Kube-Proxy for High Performance
> In standard Service networking, packets go through `kube-proxy` iptables DNAT rules.  
> Most modern Ingress Controllers (like `ingress-nginx`) **bypass `kube-proxy` entirely**. When an Ingress rule points to `api-service`, the controller reads the `Endpoints` / `EndpointSlice` object of that Service directly from the API and routes HTTP traffic **directly to the backend Pod IPs**. This avoids unnecessary network hops and preserves client source IPs.

---

## 4. Ingress Routing Types & Declarative YAML Definitions

Kubernetes Ingress supports two primary routing strategies: **Path-based routing** and **Host-based routing**.

### Type 1: Single Service Ingress (Simple Proxy)
Routes all incoming traffic regardless of host or path to a single backend Service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: web-service
      port:
        number: 80
```

---

### Type 2: Path-Based Routing (Routing by URL Path)
Routes traffic arriving at the same domain to different backend microservices based on the request URL path (e.g., `/app` vs `/api`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-api-service
                port:
                  number: 8080
```

---

### Type 3: Host-Based Routing (Virtual Hosting)
Routes traffic arriving at different domain names (via DNS matching) to completely independent backend Services sharing the same Ingress IP:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: store-service
                port:
                  number: 80
    - host: blog.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: blog-service
                port:
                  number: 80
```

---

### Understanding `pathType`: Exact vs. Prefix vs. ImplementationSpecific

In Kubernetes Ingress v1 (`networking.k8s.io/v1`), the `pathType` field is **mandatory**:

* **`Exact`:** Matches the URL path exactly, case-sensitively.  
  * Rule: `/foo`
  * Matches: `/foo`
  * Does NOT match: `/foo/`, `/foobar`
* **`Prefix`:** Matches based on URL path prefixes split by `/`.
  * Rule: `/foo`
  * Matches: `/foo`, `/foo/`, `/foo/bar`
  * Does NOT match: `/foobar`
* **`ImplementationSpecific`:** Matching logic depends entirely on the specific Ingress Controller (e.g., regex matching in NGINX).

---

## 5. TLS / HTTPS Termination in Ingress

To secure Ingress with HTTPS:
1. Store the TLS private key and certificate in a Kubernetes Secret of type `kubernetes.io/tls`.
2. Reference the Secret under `spec.tls` in the Ingress manifest.

### Step 1: Create the TLS Secret
```bash
# Imperative command:
kubectl create secret tls webapp-tls \
  --cert=/path/to/tls.crt \
  --key=/path/to/tls.key
```

### Step 2: Reference Secret in Ingress YAML
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - secure.example.com
      secretName: webapp-tls
  rules:
    - host: secure.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-service
                port:
                  number: 443
```

The Ingress Controller automatically terminates the TLS handshake on port 443, decrypts the payload, and forwards plain HTTP traffic to the backend Service over the private cluster network.

---

## 6. Annotations & URL Rewriting (NGINX Rewrite Target)

When an application is hosted at the root path (`/`) inside its container, but Ingress routes external traffic to it via a subpath like `/pay`, the backend container receives requests starting with `/pay` and returns an HTTP 404.

To strip or rewrite the path before sending traffic to the backend Pod, use Ingress annotations:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /pay
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 80
```
* **Effect:** When a client requests `http://<ip>/pay`, the Ingress Controller rewrites the path to `/` before forwarding the packet to `payment-service`.

---

## 7. Fast Imperative Commands for CKA

Under exam conditions, typing long Ingress YAML files from scratch consumes too much time. Use `kubectl create ingress` with `--dry-run=client -o yaml` to generate templates in seconds:

```bash
# 1. Simple Single-Rule Ingress:
kubectl create ingress simple-ing --rule="myapp.com/=my-svc:80"

# 2. Path-based Ingress with multiple paths:
kubectl create ingress path-ing \
  --rule="app.com/pay=pay-svc:80" \
  --rule="app.com/orders=order-svc:8080"

# 3. Host-based Ingress:
kubectl create ingress host-ing \
  --rule="red.app.com/*=red-svc:80" \
  --rule="blue.app.com/*=blue-svc:80"

# 4. Generate YAML template to add annotations or TLS:
kubectl create ingress my-ing --rule="app.com/api=api-svc:8080" --dry-run=client -o yaml > ing.yaml
```

---

## 8. Summary Comparison Cheat Sheet

| Feature | Service | Ingress Controller | Ingress Resource |
| :--- | :--- | :--- | :--- |
| **Kubernetes Kind** | `kind: Service` | `kind: Deployment` / `DaemonSet` | `kind: Ingress` |
| **Role** | L4 abstraction & internal routing | Active L7 reverse proxy daemon | Routing policy & configuration rulebook |
| **Direct Client Entry?** | Only if `NodePort` or `LoadBalancer` | Yes (exposed via NodePort / Cloud LB) | No (declarative spec only) |
| **Path Rewriting?** | No | Yes (via annotations / regex) | Defined via metadata annotations |
| **TLS Offloading?** | No | Yes (reads K8s TLS secrets) | Declared under `spec.tls` |

---

## 9. Ingress Walkthrough: YAML Examples and Where Port Numbers Come From

### 9.1 Where Does the Port Number Come From?

A port number is defined in several places. Each layer only has to match the layer it talks to:

```text
User ──► Ingress Controller (80 / 443) ──► Service (port: 8080) ──► Pod (targetPort: 8080)
                                                                        │
                                                        Application listens on 8080
```

| Layer | Field | Required? | Purpose |
| :--- | :--- | :--- | :--- |
| Application code | e.g. `app.listen(8080)` | **Yes** | The process must listen on this port |
| Dockerfile | `EXPOSE 8080` | No | Documentation only. It does **not** open or publish the port |
| Deployment (pod spec) | `containerPort: 8080` | No | Informational. Traffic is not blocked without it |
| Service | `port` and `targetPort` | **Yes** | `port` is the Service's port; `targetPort` is the Pod's port |
| Ingress | `backend.service.port.number` | **Yes** | Must match the Service's `port` |

**Dockerfile (documentation only):**

```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 8080
CMD ["node", "server.js"]
```

**Deployment (pod spec):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wear-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wear
  template:
    metadata:
      labels:
        app: wear
    spec:
      containers:
      - name: wear
        image: myrepo/wear:1.0
        ports:
        - containerPort: 8080
```

**Service (the port that actually connects things):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wear-service
spec:
  selector:
    app: wear
  ports:
  - port: 8080        # Port of the Service (used by Ingress)
    targetPort: 8080  # Port of the Pod / application
```

**Rules to remember:**

- `targetPort` must equal the port the application really listens on.
- The Ingress `port.number` must equal the Service's `port`, not its `targetPort`. You can also reference a named port with `port.name`.
- End users never type `8080`. They connect to the Ingress Controller on port 80 or 443.
- With a `NodePort` Service, a `nodePort` (30000–32767) is also assigned. With Ingress, you usually do not need to set it.

---

### 9.2 Ingress Examples

Each example below uses the Services from the walkthrough. Their `port` values are the ones the Ingress points to.

#### Example 1: Single Service (Default Backend)

All traffic goes to `wear-service`, regardless of host or path:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: wear-service
      port:
        number: 8080
```

```bash
kubectl create -f ingress-wear.yaml
kubectl get ingress
```

#### Example 2: Path-Based Routing

One domain, two paths, two Services. Requests that match neither path fall through to the default backend:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-paths
spec:
  ingressClassName: nginx
  rules:
  - host: myonlinestore.com
    http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 8080
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: video-service
            port:
              number: 8080
```

To send unmatched URLs to a 404 page service, add a `defaultBackend` under `spec`. You must deploy that service in the cluster yourself:

```yaml
spec:
  defaultBackend:
    service:
      name: default-http-backend
      port:
        number: 80
```

#### Example 3: Host-Based Routing

Each subdomain gets its own rule. DNS records for `wear.myonlinestore.com` and `watch.myonlinestore.com` must point to the Ingress Controller's external IP:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-hosts
spec:
  ingressClassName: nginx
  rules:
  - host: wear.myonlinestore.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 8080
  - host: watch.myonlinestore.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: video-service
            port:
              number: 8080
```

#### Example 4: Host and Path Combined

Each host is split again by path:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-combined
spec:
  ingressClassName: nginx
  rules:
  - host: wear.myonlinestore.com
    http:
      paths:
      - path: /shopping
        pathType: Prefix
        backend:
          service:
            name: shopping-service
            port:
              number: 8080
      - path: /exchange
        pathType: Prefix
        backend:
          service:
            name: exchange-service
            port:
              number: 8080
      - path: /support
        pathType: Prefix
        backend:
          service:
            name: support-service
            port:
              number: 8080
  - host: watch.myonlinestore.com
    http:
      paths:
      - path: /movies
        pathType: Prefix
        backend:
          service:
            name: movies-service
            port:
              number: 8080
      - path: /tv
        pathType: Prefix
        backend:
          service:
            name: tv-service
            port:
              number: 8080
```

#### Example 5: HTTPS (TLS)

Create the certificate Secret first, then reference it under `spec.tls`:

```bash
kubectl create secret tls myonlinestore-tls \
  --cert=tls.crt \
  --key=tls.key
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-secure
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myonlinestore.com
    secretName: myonlinestore-tls
  rules:
  - host: myonlinestore.com
    http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 8080
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: video-service
            port:
              number: 8080
```

The Ingress Controller terminates TLS, so the applications only handle plain HTTP inside the cluster.

---

### 9.3 Quick Checklist

- Ingress needs an Ingress Controller installed in the cluster. A bare cluster ignores Ingress objects.
- `pathType` is required in `networking.k8s.io/v1`. Common values are `Prefix` and `Exact`.
- Ingress `port.number` = Service `port`. Service `targetPort` = application port.
- The Dockerfile `EXPOSE` and the pod's `containerPort` do not open or route anything by themselves.
