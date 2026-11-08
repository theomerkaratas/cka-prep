To verify which port a Kubernetes control plane or worker component is listening on directly at the OS level, query the kernel socket table using `ss` or `netstat`.

---

## 1. Socket Inspection Flags

When executing socket inspection utilities, combine these flags:

- **`-n` (numeric):** Displays raw numeric IP addresses and port numbers instead of resolving service names (much faster).
- **`-p` (process):** Shows the process name and PID responsible for opening the socket.
- **`-l` (listening):** Filters only for sockets in the `LISTEN` state.
- **`-t` (tcp):** Restricts the output to TCP sockets.

---

## 2. Diagnostic Commands

```bash
# Modern and fast socket inspection tool (ss):
ss -nplt | grep kube-sched
ss -nplt | grep etcd
ss -nplt | grep kube-api
ss -nplt | grep kubelet

# Traditional utility (netstat):
netstat -nplt | grep kube-scheduler
netstat -nplt | grep 6443
```

---

## 3. Sample Output Breakdown

```text
LISTEN 0 4096 127.0.0.1:10259 0.0.0.0:* users:(("kube-scheduler",pid=1234,fd=7))
```

- **`127.0.0.1:10259`:** The bound IP address and listening **Port Number (`10259`)**.
- **`users:(("kube-scheduler",...))`:** Confirms the process owning this listening port.

---

## 4. Alternative Discovery Methods for Kubernetes Components

1. **Static Pod Manifests:**  
    Inspect the YAML definitions in `/etc/kubernetes/manifests/` (e.g., `cat /etc/kubernetes/manifests/kube-scheduler.yaml` and look for `--secure-port=...`).
2. **Process Arguments:**  
    Query the process table: `ps -ef | grep kube-scheduler`.
3. **Kubectl Describe:**  
    Query the pod spec: `kubectl describe pod kube-scheduler-controlplane -n kube-system`.

---

## 5. Master CKA Port Reference Sheet

|Port / Range|Component|Direction & Scope|
|---|---|---|
|**`6443`**|`kube-apiserver`|Inbound from all worker nodes, administrators, and CI/CD pipelines.|
|**`2379`**|`etcd` Client API|Inbound from `kube-apiserver` only.|
|**`2380`**|`etcd` Peer Communication|Inbound between control plane nodes (HA consensus).|
|**`10259`**|`kube-scheduler`|Localhost / Control plane internal health endpoint.|
|**`10257`**|`kube-controller-manager`|Localhost / Control plane internal health endpoint.|
|**`10250`**|`kubelet` API|Inbound from `kube-apiserver` for `kubectl logs` and `kubectl exec`.|
|**`30000 - 32767`**|NodePort Services|Inbound from external clients / load balancers to worker nodes.|