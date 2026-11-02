# Service Networking in the CKA Exam: Common Scenarios and Simple Troubleshooting

In the **CKA (Certified Kubernetes Administrator)** exam, Service networking questions usually focus on two things:
1. **Inspecting cluster-level Service configurations** (e.g., finding the Service CIDR range or proxy mode).
2. **Troubleshooting Service connectivity issues** (e.g., discovering why a Service cannot route traffic to its backend Pods).

Here are the key scenarios and simple, easy-to-remember commands.

---

## Quick Location Reference

| What you want to check | Where to look |
| :--- | :--- |
| **Service ClusterIP CIDR** | `/etc/kubernetes/manifests/kube-apiserver.yaml` (`--service-cluster-ip-range`) |
| **Pod Network CIDR** | `/etc/kubernetes/manifests/kube-controller-manager.yaml` (`--cluster-cidr`) |
| **`kube-proxy` Configuration** | ConfigMap `kube-proxy` in namespace `kube-system` |
| **`kube-proxy` Logs** | Pod logs in namespace `kube-system` with label `k8s-app=kube-proxy` |

---

## Scenario 1: Identify the Service and Pod CIDR Ranges

### Exam Task A
> *"What is the IP range configured for the services within the cluster?"*

#### Simple Command:
Run this on the control plane node to check `kube-apiserver`:
```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
```

*Example output:*
```text
- --service-cluster-ip-range=10.96.0.0/12
```
* **Answer:** `10.96.0.0/12`

---

### Exam Task B
> *"What is the range of IP addresses configured for pods on the cluster?"*

#### Option 1 (Fastest): Check `kube-controller-manager` manifest
Run this on the control plane node:
```bash
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr
```

*Example output:*
```text
- --cluster-cidr=10.244.0.0/16
```
* **Answer:** `10.244.0.0/16`

#### Option 2: Check `kube-proxy` ConfigMap
If you only have `kubectl` access (no direct file access on control plane):
```bash
kubectl describe cm kube-proxy -n kube-system | grep clusterCIDR
```

#### Option 3: Check a Node's `podCIDR`
If you want to see the specific subnet slice given to a single node:
```bash
kubectl describe node <node-name> | grep -i cidr
# or:
kubectl get node <node-name> -o jsonpath='{.spec.podCIDR}'
```

---

## Scenario 2: Find the Proxy Mode (Type of Proxy) used by `kube-proxy`

### Exam Task
> *"What type of proxy is the `kube-proxy` configured to use in this cluster?"*  
> *(or: "What is the proxy mode (iptables or ipvs) configured for kube-proxy?")*

### Option 1 (Fastest & Guaranteed): Check `kube-proxy` Pod Logs
When `kube-proxy` starts up, it logs the exact proxy mode it initialized.

```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep -i using
```

* **Output if iptables:**
  ```text
  Using iptables Proxier.
  ```
  **Answer:** `iptables`

* **Output if IPVS:**
  ```text
  Using ipvs Proxier.
  ```
  **Answer:** `ipvs`

> **Alternative step-by-step:** If you prefer checking a specific pod:
> ```bash
> kubectl get pods -n kube-system | grep kube-proxy
> kubectl logs <kube-proxy-pod-name> -n kube-system | grep -i using
> ```

---

### Option 2: Check the `kube-proxy` ConfigMap
`kube-proxy` reads its settings from a ConfigMap in the `kube-system` namespace.

```bash
kubectl describe cm kube-proxy -n kube-system | grep -i mode
```

*Example output:*
```text
mode: "iptables"
```
*or:*
```text
mode: "ipvs"
```

> [!IMPORTANT]
> **What if `mode: ""` is empty?**  
> If the value is empty (`""`), it means the cluster is using the Kubernetes default mode, which is **`iptables`**. Your answer is `iptables`.

---

## Scenario 3: Service Has No Endpoints (Pods Not Receiving Traffic)

This is the **most common troubleshooting scenario** in the CKA exam. A Service exists, but connecting to it fails with a timeout or connection refused.

### Step 1: Check the Service and its Endpoints
```bash
kubectl get svc <service-name>
kubectl get endpoints <service-name>
```

* **Broken Output:**
  ```text
  NAME         ENDPOINTS
  my-service   <none>
  ```
  If `ENDPOINTS` is `<none>`, traffic cannot be routed to any Pod!

### Step 2: Compare Selectors and Labels
1. Check the selector configured on the Service:
   ```bash
   kubectl describe svc <service-name> | grep -i selector
   ```
   *Example:* `Selector: app=web-app`

2. Check the actual labels on your Pods:
   ```bash
   kubectl get pods --show-labels
   ```
   If the Pods have `app=webapp` (without hyphen) while the Service has `app=web-app`, the selector does not match.

### Step 3: Fix the Selector
Edit the service and update the selector to match the Pod label:
```bash
kubectl edit svc <service-name>
```
Once saved, re-check:
```bash
kubectl get endpoints <service-name>
```
You should now see the Pod IPs listed under `ENDPOINTS`.

---

## Scenario 4: TargetPort Mismatch

Another common CKA issue is a misconfigured port mapping.

### Step 1: Describe the Service and target Pod
```bash
kubectl describe svc <service-name>
```
Look at the port configuration:
```text
Port:          80/TCP
TargetPort:    8080/TCP
```

### Step 2: Check the Pod's listening container port
```bash
kubectl describe pod <pod-name> | grep -i port
```
* If the container is listening on port `80`, but the Service's `TargetPort` is set to `8080`, packets will be dropped by the container.

### Step 3: Fix the TargetPort
```bash
kubectl edit svc <service-name>
```
Set `targetPort` to match the port the application is actually listening on.

---

## Scenario 5: Testing Service Connectivity from Inside the Cluster

In the exam, to verify whether a Service is reachable, spin up a temporary debug pod.

### Run a quick curl test:
```bash
kubectl run test-pod --image=busybox:1.28 --rm -it --restart=Never -- sh
```

Inside the test pod, run:
```sh
# 1. Test using Service ClusterIP:
wget -O- 10.96.0.10:80

# 2. Test using Service DNS name:
wget -O- my-service.default.svc.cluster.local:80
wget -O- my-service:80
```

---

## Scenario 6: Find a Node's Primary Interface, IP Address, and Subnet Mask

### Exam Task
> *"What is the IP address and subnet mask assigned to the primary interface of `node01`?"*

A node's **primary interface** is the network device that routes traffic to the **default gateway**.

### Step 1: Switch to the node
```bash
ssh node01
```

### Step 2: Identify the primary interface name
```bash
ip route show default
```
*Example output:*
```text
default via 192.168.1.1 dev eth0 proto dhcp metric 100
```
Look at the word right after **`dev`**. In this example, the primary interface is **`eth0`** (it could also be `ens3`, `enp0s3`, etc.).

### Step 3: Check IP address and subnet mask of that interface
```bash
ip a show eth0
```
*Example output:*
```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0
```
Inspect the **`inet`** line:
* **IP Address:** `192.168.1.10`
* **Subnet Mask (CIDR prefix):** `/24`

### Quick Subnet Mask Conversion Reference:
| CIDR | Subnet Mask (Dotted Decimal) |
| :--- | :--- |
| `/8` | `255.0.0.0` |
| `/16` | `255.255.0.0` |
| `/24` | `255.255.255.0` |
| `/20` | `255.255.240.0` |

---

## Summary: Simple Command Workflow for CKA

```bash
# 1. Find Node's Primary Interface & IP/Mask:
ip route show default     # Find interface name after 'dev'
ip a show <interface>     # Read IP and subnet mask on 'inet' line

# 2. Check Service IP and Endpoints:
kubectl get svc <name>
kubectl get endpoints <name>

# 3. Check if Pod labels match Service selector:
kubectl describe svc <name> | grep Selector
kubectl get pods --show-labels

# 4. Check Service & Pod CIDRs on Control Plane:
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr

# 5. Check kube-proxy mode:
kubectl describe cm kube-proxy -n kube-system | grep -i mode
```

