# Verifying Ingress with curl in the CKA Exam

In the exam you rarely have a browser. Use `curl` from the terminal to check that an Ingress routes traffic correctly.

---

## Step 1: Find the Address

```bash
kubectl get ingress <name>
```

Use the `ADDRESS` value if it is filled in. If it is empty, check the Ingress Controller Service:

```bash
kubectl get svc -n ingress-nginx
```

* `LoadBalancer`: use the `EXTERNAL-IP`.
* `NodePort`: use `<node-ip>:<nodeport>`.

---

## Step 2: Send a Request with the Host Header

Ingress routes by host name, so always send the `Host` header:

```bash
curl -H "Host: <host>" http://<address>/<path>
```

If DNS is not set up, resolve the host name directly:

```bash
curl --resolve <host>:80:<address> http://<host>/<path>
```

---

## Step 3: Read the Result

Show only the HTTP status code:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: <host>" http://<address>/<path>
```

Show the response body:

```bash
curl -s -H "Host: <host>" http://<address>/<path>
```

Show the status code and headers:

```bash
curl -i -H "Host: <host>" http://<address>/<path>
```

### Interpreting the Status Code

| Result | Meaning | Next check |
| :--- | :--- | :--- |
| `200` and the expected content | Routing works | None |
| `404` with `default backend - 404` in the body | No rule matched and there is no `defaultBackend` | `kubectl describe ingress <name>` |
| `404` with the application's own page | The request reached the backend, but the path does not exist there | Check `rewrite-target` and the app path |
| `502` or `503` | The Service has no ready endpoints or the backend is unreachable | `kubectl get endpoints <svc>` and `kubectl get pods` |

---

## Useful Flags

| Flag | Purpose |
| :--- | :--- |
| `-s` | Silent mode, no progress output |
| `-i` | Include response headers |
| `-o <file>` | Save the body to a file |
| `-w "%{http_code}"` | Print only the status code |
| `-H "Host: ..."` | Set the host name for Ingress routing |
| `-L` | Follow redirects |
| `-k` | Skip TLS certificate verification (self-signed certificates) |

### Combined: `-kL` for HTTPS

For an Ingress with TLS and a self-signed certificate, use `-k` and `-L` together:

```bash
curl -kL -H "Host: <host>" https://<address>/<path>
```


---

## Quick Check Workflow

```bash
# 1. Get the address
kubectl get ingress <name>

# 2. Check the status code for each path
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: <host>" http://<address>/<path1>
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: <host>" http://<address>/<path2>

# 3. Confirm the body comes from the expected backend
curl -s -H "Host: <host>" http://<address>/<path1>

# 4. If the result is wrong, inspect the Ingress and its backends
kubectl describe ingress <name>
kubectl get endpoints <svc-name>
```
