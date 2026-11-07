# CoreDNS in Kubernetes for the CKA Exam: Common Scenarios and Simple Troubleshooting

In the **CKA (Certified Kubernetes Administrator)** exam, CoreDNS tasks typically involve:

1. **Locating CoreDNS configuration files and ConfigMaps.**
2. **Troubleshooting failing or looping CoreDNS pods.**
3. **Modifying the Corefile to configure forwarding or upstream DNS.**
4. **Fixing Kubelet DNS configuration.**

Here are the key scenarios and simple, easy-to-remember commands.

---

## Quick Location Reference

| Resource                         | Where to find it                                |
| :------------------------------- | :---------------------------------------------- |
| **CoreDNS Deployment**           | `kubectl get deployment coredns -n kube-system` |
| **CoreDNS ConfigMap**            | `kubectl get cm coredns -n kube-system -o yaml` |
| **CoreDNS Service (`kube-dns`)** | `kubectl get svc kube-dns -n kube-system`       |
| **Kubelet ClusterDNS Config**    | `/var/lib/kubelet/config.yaml` on worker nodes  |

---

## Scenario 1: Locate and Inspect the CoreDNS `Corefile`

### Exam Task

> _"Where is the CoreDNS configuration stored, and what is the configured cluster domain name?"_

### Step 1: View the CoreDNS ConfigMap

```bash
kubectl describe cm coredns -n kube-system
```

Look at the `Corefile` section:

```corefile
kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure
    fallthrough in-addr.arpa ip6.arpa
    ttl 30
}
```

- **Storage:** Stored in Kubernetes ConfigMap `coredns` in namespace `kube-system`.
- **Cluster Domain:** `cluster.local`.

---

## Scenario 2: CoreDNS Pods in `CrashLoopBackOff` (Infinite Forwarding Loop)

A classic CKA / Killer.sh scenario: CoreDNS pods keep crashing every few seconds.

### Step 1: Check CoreDNS logs

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

Look for an error like:

```text
plugin/loop: Loop (127.0.0.1:53 -> :53) detected for zone "."
```

### Step 2: Why does this happen?

The `Corefile` has `forward . /etc/resolv.conf`. If the host node's `/etc/resolv.conf` contains `nameserver 127.0.0.1` (or points back to CoreDNS), queries loop indefinitely until CoreDNS crashes to protect itself.

### Step 3: Fix

Edit the CoreDNS ConfigMap to forward directly to an upstream public or corporate DNS IP (e.g. `8.8.8.8`):

```bash
kubectl edit cm coredns -n kube-system
```

Change:

```corefile
forward . /etc/resolv.conf
```

To:

```corefile
forward . 8.8.8.8
```

Restart the CoreDNS deployment to apply changes immediately:

```bash
kubectl rollout restart deployment coredns -n kube-system
```

---

## Scenario 3: Pods Cannot Resolve Cluster Services (Kubelet Misconfiguration)

### Exam Task

> _"Pods created on `node01` cannot resolve any service names. Investigate and fix the issue on `node01`."_

### Step 1: Check `/etc/resolv.conf` inside a broken pod

```bash
kubectl exec <failing-pod> -- cat /etc/resolv.conf
```

- If the `nameserver` IP is missing or wrong (e.g., `10.96.0.99` instead of `10.96.0.10`), the pod cannot reach CoreDNS.

### Step 2: Check the true `kube-dns` Service IP

```bash
kubectl get svc -n kube-system kube-dns
```

Note the `CLUSTER-IP` (e.g., `10.96.0.10`).

### Step 3: Check Kubelet configuration on `node01`

```bash
ssh node01
cat /var/lib/kubelet/config.yaml | grep -A 2 clusterDNS
```

_Example broken output:_

```yaml
clusterDNS:
  - 10.96.0.99 # <--- WRONG IP!
```

### Step 4: Fix and restart Kubelet

1. Edit the file:
   ```bash
   sudo vi /var/lib/kubelet/config.yaml
   ```
2. Change the IP under `clusterDNS` to match the `kube-dns` service IP (`10.96.0.10`).
3. Restart Kubelet:
   ```bash
   sudo systemctl restart kubelet
   ```
   _(Note: Newly created pods on this node will now receive the correct DNS server IP)._

---

## Scenario 4: CoreDNS Pods Stuck in `Pending`

### Exam Task

> _"CoreDNS pods are in Pending state. Why?"_

### Step 1: Describe the CoreDNS pod

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl describe pod <coredns-pod-name> -n kube-system
```

Look at the **Events** section at the bottom:

- **Common Cause 1: CNI Not Installed.**  
  If the output says `0/1 nodes are available: 1 node(s) had untolerated taint {node.kubernetes.io/not-ready: }`, the worker nodes are `NotReady` because a CNI plugin (like Flannel or Calico) has not been installed yet.
- **Common Cause 2: Master Node Taint.**  
  In single-node clusters, if the control-plane node has `node-role.kubernetes.io/control-plane:NoSchedule`, CoreDNS pods cannot be scheduled unless untainted.

---

## Scenario 5: What is the CoreDNS Server IP that should be configured on Pods?

### Exam Task

> _"What is the IP address of the CoreDNS server that should be configured on pods to resolve services?"_  
> _(or: "What nameserver IP should be configured in `/etc/resolv.conf` of application pods?")_

### ⚠️ The Trap to Avoid:

Do **NOT** inspect the individual CoreDNS Pod IPs (`kubectl get pods -o wide`)! Pod IPs are dynamic and change whenever a pod restarts.  
Pods never talk directly to CoreDNS Pod IPs; they communicate via its stable front-end Service: **`kube-dns`**.

---

### Method 1 (Direct & Fastest): Check the `kube-dns` Service IP

Run this command from your terminal:

```bash
kubectl get svc -n kube-system kube-dns
```

_Example output:_

```text
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   10d
```

- **Answer:** `10.96.0.10` (The `CLUSTER-IP` of `kube-dns`).

---

### Method 2: Check any existing Pod's `/etc/resolv.conf`

If any pod is already running in the cluster, its `nameserver` line will directly tell you the answer:

```bash
kubectl exec <any-running-pod> -- cat /etc/resolv.conf
```

_Example output:_

```ini
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- **Answer:** `10.96.0.10` (The `nameserver` entry).

---

### Method 3: Check Kubelet's config on a Node

Kubelet is the component that injects this IP into every new pod:

```bash
cat /var/lib/kubelet/config.yaml | grep -A 2 clusterDNS
```

_Example output:_

```yaml
clusterDNS:
  - 10.96.0.10
```

- **Answer:** `10.96.0.10`

---

## Scenario 6: Where is the configuration file located inside the CoreDNS Pod?

### Exam Task

> _"Where is the configuration file located inside the CoreDNS pod?"_  
> _(or: "What is the path of the Corefile inside the coredns container?")_

### Quick Fact:

In standard Kubernetes, the CoreDNS ConfigMap is mounted at:
👉 **`/etc/coredns/Corefile`**

---

### Method 1: Using `kubectl describe deployment`

Inspect the Deployment in `kube-system`:

```bash
kubectl describe deployment coredns -n kube-system | grep -A 3 -i "mounts:"
```

_Example output:_

```text
    Mounts:
      /etc/coredns from config-volume (ro)
```

Or check the container startup arguments:

```bash
kubectl describe deployment coredns -n kube-system | grep -i "corefile"
# Args: -conf /etc/coredns/Corefile
```

---

### Method 2: Using `-o yaml`

Export the Deployment YAML and check `volumeMounts`:

```bash
kubectl get deployment coredns -n kube-system -o yaml | grep -A 2 -i volumeMounts
```

_Example output:_

```yaml
volumeMounts:
  - mountPath: /etc/coredns
    name: config-volume
    readOnly: true
```

Combined with the ConfigMap key (`Corefile`), the path is:

- **Directory:** `/etc/coredns`
- **Full File Path:** `/etc/coredns/Corefile`

---

## Summary: Simple Command Workflow for CKA

```bash
# 1. View CoreDNS ConfigMap & Corefile:
kubectl describe cm coredns -n kube-system

# 2. Check CoreDNS pod status & logs:
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=30

# 3. Check kube-dns Service ClusterIP & Endpoints:
kubectl get svc -n kube-system kube-dns
kubectl get endpoints -n kube-system kube-dns

# 4. Check Kubelet DNS config on worker node:
cat /var/lib/kubelet/config.yaml | grep -A 2 clusterDNS
```
