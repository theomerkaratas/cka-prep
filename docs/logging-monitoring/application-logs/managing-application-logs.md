# Managing Application Logs

Kubernetes captures output written to standard output (`stdout`) and standard error (`stderr`) streams by containers and stores them on the host node (typically under `/var/log/pods/`).

You can inspect these logs using `kubectl logs`.

---

## 1. Single-Container Pod Logs

```bash
# View logs of a pod
kubectl logs pod-name -n namespace-name

# Stream live logs in real time (equivalent to tail -f)
kubectl logs -f pod-name

# View only the last N lines
kubectl logs --tail=50 pod-name

# Stream the last 100 lines continuously
kubectl logs --tail=100 -f pod-name

# View logs generated within the last time duration
kubectl logs --since=1h pod-name
kubectl logs --since=15m pod-name
```

---

## 2. Multi-Container Pod Logs

If a pod has more than one container, you **must specify the target container name** using the `-c` flag:

```bash
# View logs from a specific container inside a multi-container pod
kubectl logs pod-name -c container-name -n namespace-name

# Stream logs from a specific container
kubectl logs -f pod-name -c container-name
```

---

## 3. Troubleshooting Crashed Pods (`--previous`)

When a container crashes or is restarted (in a `CrashLoopBackOff` state), running standard `kubectl logs` only displays logs from the *newly started* container instance.

To view logs from the **previously terminated container instance** that caused the crash:

```bash
# Inspect logs from the previous failed container instance
kubectl logs pod-name --previous
# or shorthand:
kubectl logs pod-name -p
```

---

## 4. Querying Logs across Multiple Pods

You can aggregate logs from all pods matching a specific label selector:

```bash
# Stream logs from all pods with label app=frontend
kubectl logs -l app=frontend --tail=20 -f

# Include container name prefix when querying across multiple pods
kubectl logs -l app=frontend --prefix=true
```
 