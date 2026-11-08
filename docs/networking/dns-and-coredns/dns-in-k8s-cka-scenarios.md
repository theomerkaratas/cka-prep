# DNS in Kubernetes for the CKA Exam: Common Scenarios and Simple Troubleshooting

In the **CKA (Certified Kubernetes Administrator)** exam, DNS questions typically involve two tasks:
1. **Verifying DNS records** (identifying Service FQDNs or testing resolution from a pod).
2. **Troubleshooting DNS failures** (e.g., CoreDNS pods down, Service unreachable by name, or `/etc/resolv.conf` misconfigured).

Here are the key scenarios and simple, easy-to-remember commands.

---

## Quick FQDN Cheat Sheet

Remember the standard naming formats:

| Target | FQDN Format | Example |
| :--- | :--- | :--- |
| **Service** | `<svc-name>.<namespace>.svc.cluster.local` | `web-service.default.svc.cluster.local` |
| **Pod (Default)** | `<dashed-ip>.<namespace>.pod.cluster.local` | `10-244-1-5.default.pod.cluster.local` |
| **Pod (Headless)** | `<pod-hostname>.<headless-svc>.<namespace>.svc.cluster.local` | `db-0.db-service.default.svc.cluster.local` |

---

## Scenario 1: Test DNS Resolution from Inside the Cluster

### Exam Task
> *"Verify if the service `nginx-service` in namespace `prod` is resolvable via DNS from the `default` namespace."*

### Fast 1-liner test (No permanent pod needed):
```bash
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-service.prod
```

* **If it resolves:** You will see the Service's ClusterIP:
  ```text
  Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
  Name:      nginx-service.prod.svc.cluster.local
  Address 1: 10.100.200.5 nginx-service.prod.svc.cluster.local
  ```
* **If it fails:** It returns `server can't find nginx-service.prod: NXDOMAIN`.

> **Exam Tip:** Always use **`busybox:1.28`** in CKA exams. Newer versions of busybox (`latest`) sometimes have an `nslookup` bug that fails on relative search paths.

---

## Scenario 2: Troubleshooting CoreDNS Issues

If all DNS queries inside the cluster fail with timeouts:

### Step 1: Check CoreDNS Pods status
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```
* **What to look for:**
  - Are CoreDNS pods in **`Running`** state with `1/1` ready?
  - If in `Pending`: There is a scheduling/taint issue or CNI is not installed.
  - If in `CrashLoopBackOff`: Check logs in Step 2.

### Step 2: Check CoreDNS Logs
```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=20    # (kube-proxy)
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50      # (CoreDNS)
```

### Step 3: Check CoreDNS Service (`kube-dns`)
Pods send queries to the `kube-dns` Service IP. If this Service has no endpoints, DNS fails completely:
```bash
kubectl get svc -n kube-system kube-dns
kubectl get endpoints -n kube-system kube-dns
```
* Under `ENDPOINTS`, you should see the IP addresses of the CoreDNS pods (e.g. `10.244.0.3:53, 10.244.0.4:53`).

---

## Scenario 3: Inspect a Pod's `/etc/resolv.conf`

### Exam Task
> *"Check the nameserver and search domains configured for pod `webapp`."*

### Simple Command:
```bash
kubectl exec webapp -- cat /etc/resolv.conf
```

*Example output:*
```ini
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

* **`nameserver`:** Must match the `CLUSTER-IP` of `kubectl get svc -n kube-system kube-dns`.
* **`search`:** The search paths appended automatically when you curl a short name like `curl web-service`.

---

## Scenario 4: Creating a Headless Service for Stateful Pods

### Exam Task
> *"Create a headless service named `db-svc` on port 3306 that targets pods with label `app=db`."*

A **Headless Service** is just a standard Service with **`clusterIP: None`**.

### Step 1: Generate YAML using `--dry-run`
```bash
kubectl create service clusterip db-svc --tcp=3306:3306 --dry-run=client -o yaml > db-svc.yaml
```

### Step 2: Edit YAML and set `clusterIP: None`
Open the generated file:
```bash
vi db-svc.yaml
```

Make it look like this:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-svc
spec:
  clusterIP: None       # <--- ADD THIS LINE
  selector:
    app: db
  ports:
    - port: 3306
      targetPort: 3306
```

### Step 3: Apply the file
```bash
kubectl apply -f db-svc.yaml
```

Verify that ClusterIP is `None`:
```bash
kubectl get svc db-svc
# NAME     TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# db-svc   ClusterIP   None         <none>        3306/TCP
```

---

## Summary: Simple Command Workflow for CKA

```bash
# 1. Quick DNS lookup test:
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup <service-name>

# 2. Check CoreDNS status & endpoints:
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get endpoints -n kube-system kube-dns

# 3. Read pod's resolver configuration:
kubectl exec <pod-name> -- cat /etc/resolv.conf

# 4. Create Headless Service:
kubectl create service clusterip <name> --tcp=<port>:<targetPort> --dry-run=client -o yaml
# (then set clusterIP: None inside the YAML)
```

