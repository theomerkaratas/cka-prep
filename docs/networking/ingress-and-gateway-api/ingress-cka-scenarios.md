# Ingress in the CKA Exam: Common Scenarios and Simple Troubleshooting

In the **CKA (Certified Kubernetes Administrator)** exam, Ingress tasks are very popular and generally fall into four categories:
1. **Creating an Ingress resource quickly using imperative commands.**
2. **Adding URL Rewrite annotations (fixing HTTP 404 on backend services).**
3. **Configuring TLS Termination with a Kubernetes Secret.**
4. **Troubleshooting Ingress routing errors and missing IngressClasses.**

Here are the key scenarios and simple, easy-to-remember commands.

---

## Quick Reference: `networking.k8s.io/v1` Ingress Skeleton

If you ever need to inspect or write an Ingress resource from memory:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
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
                name: app-service
                port:
                  number: 80
```

---

## Scenario 1: Fast Ingress Creation using `kubectl create ingress`

In the exam, writing Ingress YAML from scratch takes too much time. Use `kubectl create ingress` with `--dry-run=client -o yaml` to generate it in seconds.

### Exam Task A: Single Host & Path
> *"Create an Ingress named `web-ing` that routes `http://wear.my-shop.com/wear` to service `wear-service:8080`."*

```bash
kubectl create ingress web-ing \
  --rule="wear.my-shop.com/wear*=wear-service:8080"
```

### Exam Task B: Multiple Paths on the Same Host
> *"Create an Ingress named `shop-ing` routing `/wear` to `wear-svc:80` and `/watch` to `watch-svc:80` under host `shop.com`."*

```bash
kubectl create ingress shop-ing \
  --rule="shop.com/wear*=wear-svc:80" \
  --rule="shop.com/watch*=watch-svc:80"
```

### Exam Task C: Host-Based Routing (Virtual Hosts)
> *"Create an Ingress named `domain-ing` routing `red.app.com` to `red-svc:80` and `blue.app.com` to `blue-svc:80`."*

```bash
kubectl create ingress domain-ing \
  --rule="red.app.com/*=red-svc:80" \
  --rule="blue.app.com/*=blue-svc:80"
```

---

## Scenario 2: Path Rewriting (`rewrite-target: /`)

### Exam Task
> *"Traffic sent to `http://<ip>/pay` receives an HTTP 404 from the payment service. Configure Ingress so that requests to `/pay` are forwarded to `/` on the backend service."*

### Why does this happen?
The backend application in the container listens on the root path `/`. When Ingress forwards `GET /pay`, the application looks for a `/pay` route and returns 404 Not Found.

### Step 1: Edit the Ingress Resource
```bash
kubectl edit ingress <ingress-name>
```

### Step 2: Add the NGINX rewrite annotation
Under `metadata.annotations`, add:
```yaml
metadata:
  name: payment-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```
Save and exit (`:wq`). Requests to `/pay` will now have `/pay` replaced with `/` before reaching the backend container.

---

## Scenario 3: Configuring TLS / HTTPS Termination

### Exam Task
> *"Create a Secret named `app-tls-secret` using `/tmp/tls.crt` and `/tmp/tls.key`. Then configure Ingress `secure-ing` to terminate TLS for host `secure.example.com`."*

### Step 1: Create the TLS Secret (Imperative)
```bash
kubectl create secret tls app-tls-secret \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key
```

### Step 2: Add `spec.tls` to the Ingress YAML
Generate or edit the Ingress manifest:
```bash
kubectl edit ingress secure-ing
```

Add the `tls` block right under `spec`:
```yaml
spec:
  tls:
    - hosts:
        - secure.example.com
      secretName: app-tls-secret
  rules:
    - host: secure.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-svc
                port:
                  number: 443
```

---

## Scenario 4: Troubleshooting Missing IngressClass or Default Controller

### Exam Task
> *"An Ingress resource was created, but its Address field remains empty (`<none>`) and requests time out. Fix the issue."*

### Step 1: Check Ingress status
```bash
kubectl get ingress
```
*Broken Output:*
```text
NAME        CLASS    HOSTS          ADDRESS   PORTS
test-ing    <none>   test.app.com   <none>    80
```
If `ADDRESS` is empty after several seconds, no Ingress Controller is handling this resource.

### Step 2: Check available IngressClasses
```bash
kubectl get ingressclass
```
*Example output:*
```text
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       5d
```

### Step 3: Fix: Assign `ingressClassName`
1. Edit the Ingress:
   ```bash
   kubectl edit ingress test-ing
   ```
2. Add `ingressClassName: nginx` under `spec`:
   ```yaml
   spec:
     ingressClassName: nginx
     rules:
       ...
   ```
3. Re-check `kubectl get ingress`. The `ADDRESS` field will now populate with the controller's external or node IP.

---

## Scenario 5: Check Ingress Controller Pods and Logs

If Ingress routing behaves unexpectedly, always verify that the Ingress Controller is running and inspect its access/error logs:

```bash
# 1. Check Ingress Controller pod in ingress-nginx namespace:
kubectl get pods -n ingress-nginx

# 2. Check Ingress Controller Service (find external/NodePort ports):
kubectl get svc -n ingress-nginx

# 3. Stream real-time HTTP routing and error logs:
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50
```

---

## Scenario 6: Requests Without a Matching Rule (Default Backend)

### Exam Task
> *"Ingress `shop-ing` in namespace `default` routes `shop.example.com/wear` to `wear-svc:80`. Requests to any other path on `shop.example.com`, such as `/watch`, return `404 Not Found` from the Ingress controller. Configure the Ingress so that unmatched requests for `shop.example.com` are sent to the Service `default-svc` on port `80`."*

### Why does this happen?
Ingress routing first matches the **host**, then the **path**. The request `/watch` matches the host but no path rule. Without a default backend, the Ingress controller answers with its own `404 Not Found`. No Service receives the request.

### Step 1: Confirm the current routing
```bash
kubectl describe ingress shop-ing
```
* Look at the `Rules` section. Only `/wear` should be listed.
* The `Default backend` line should be `<default>`, which means no default backend is set.

Check that the target Service exists in the same namespace as the Ingress:
```bash
kubectl get svc default-svc
```

### Step 2: Add `spec.defaultBackend`
```bash
kubectl edit ingress shop-ing
```

Add the `defaultBackend` block under `spec`. Keep the existing `rules` unchanged:
```yaml
spec:
  defaultBackend:
    service:
      name: default-svc
      port:
        number: 80
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /wear
            pathType: Prefix
            backend:
              service:
                name: wear-svc
                port:
                  number: 80
```

### Step 3: Verify
```bash
kubectl describe ingress shop-ing | grep -i "default backend"
# Default backend:  default-svc:80 (...)
```

Send a request with the Host header to the Ingress controller address (use the controller's external IP or NodePort):
```bash
curl -H "Host: shop.example.com" http://<controller-ip>/watch    # served by default-svc
curl -H "Host: shop.example.com" http://<controller-ip>/wear     # served by wear-svc
```

> **Exam Tip:** `defaultBackend` applies only when no host or path rule matches. If the host matches but the path does not, the behavior depends on the controller. With ingress-nginx, you usually get a 404 unless you add a catch-all `/` rule. Check `kubectl describe ingress` first to see which case you are in.

---

## Summary: Simple Command Workflow for CKA

```bash
# 1. Create Ingress quickly:
kubectl create ingress <name> --rule="host.com/path*=svc:port"

# 2. Check Ingress and assigned IP:
kubectl get ingress
kubectl describe ingress <name>

# 3. Create TLS Secret:
kubectl create secret tls <secret-name> --cert=<path> --key=<path>

# 4. Check available IngressClasses:
kubectl get ingressclass

# 5. Check the default backend (catch-all for unmatched requests):
kubectl describe ingress <name> | grep -i "default backend"

# 6. Check Ingress Controller status & logs:
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

