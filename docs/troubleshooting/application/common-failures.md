# Application Failure: Common Failures and Fixes

### Pod status: ImagePullBackOff / ErrImagePull
- **Cause:** Wrong image name or tag, or a private registry without credentials.
- **Check:** `kubectl describe pod` → Events.
- **Fix:** `kubectl set image deploy/web web=nginx:1.25 -n <ns>` or edit the image.

### Pod status: CrashLoopBackOff
- **Cause:** The container starts and exits repeatedly.
- **Check:** `kubectl logs <pod> --previous` (the current container may have no useful output yet). Check `describe` for exit code and reason.
- **Fix:** Correct the `command`/`args`, missing env vars, bad config, or wrong DB connection string.

### Pod status: Pending
- **Cause:** Not scheduled. Usually insufficient resources, a taint, a node selector mismatch, or an unbound PVC.
- **Check:** `kubectl describe pod` → Events (for example, `FailedScheduling`).

### Pod status: CreateContainerConfigError
- **Cause:** A referenced ConfigMap or Secret does not exist, or a key is missing.
- **Check:** `kubectl describe pod` → Events.
- **Fix:** Create the object or fix the name/key reference.

### Pod is Running but not Ready (0/1)
- **Cause:** The readiness probe is failing. The Service will not send traffic to this pod.
- **Check:** `kubectl describe pod` → Readiness probe failures; check the probe path and port.

### Pod OOMKilled
- **Cause:** The container exceeded its memory limit.
- **Check:** `lastState.terminated.reason` shows `OOMKilled`.
- **Fix:** Raise `resources.limits.memory` or fix the memory leak.

### Service has no endpoints
- **Cause 1:** The Service `selector` does not match the pod labels.
- **Cause 2:** The pods are not Ready (see above).
- **Cause 3:** The Service `targetPort` does not match the container port.
- **Check:** `kubectl describe svc` and compare selector, `targetPort`, and the pod's labels and `containerPort`.
- **Fix:** Edit the Service selector or port, or fix the pod labels.

```bash
kubectl edit svc web-svc -n <ns>
```

### Service exists but app is unreachable
- **Cause:** Wrong `port`/`targetPort`, wrong `type` (e.g., ClusterIP instead of NodePort), or the app listens on a different port or interface.
- **Check:** `kubectl get svc -o wide`, `kubectl exec <pod> -- netstat -tlnp` (or `ss -tlnp` if available).

### Application runs but cannot reach the DB
- **Cause:** The connection string uses the wrong Service name or port, or the DB pod is not Ready.
- **Check:** `kubectl exec <web-pod> -- env`, then `nslookup db-svc` and test the port from the web pod.
