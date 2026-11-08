# Application Failure: Triage Commands

Commands for checking objects, logs, and connectivity while you troubleshoot an application.

---

## Overview and Status

```bash
# Overview of everything in the namespace
kubectl get all -n <ns>
kubectl get pods -n <ns> -o wide

# Pod status, restarts, and events
kubectl describe pod <pod> -n <ns>
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

## Logs

```bash
kubectl logs <pod> -n <ns>                 # current container
kubectl logs <pod> -n <ns> --previous      # last crashed container
kubectl logs <pod> -n <ns> -c <container>  # multi-container pod
kubectl logs -f <pod> -n <ns>              # follow live output
```

## Service, Selector, and Endpoints

```bash
kubectl describe svc web-svc -n <ns>       # check Selector, TargetPort, Endpoints
kubectl get endpoints web-svc -n <ns>      # empty = no matching Ready pods
kubectl get pods -n <ns> --show-labels     # compare labels with the selector
```

## Inside the Pod

```bash
kubectl exec -it <pod> -n <ns> -- sh
kubectl exec <pod> -n <ns> -- env         # verify env vars and injected config
```

## Last Termination Reason

Shows reasons such as `OOMKilled` or `Error`:

```bash
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

## Test Connectivity from Inside the Cluster

```bash
# Test a Service by name (DNS + port)
kubectl run tmp --rm -it --restart=Never --image=curlimages/curl -n <ns> -- curl -s http://web-svc:80

# Test the DB Service
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -n <ns> -- nslookup db-svc

# Quick local test without exposing the Service
kubectl port-forward svc/web-svc 8080:80 -n <ns>
```

## Test from Outside (NodePort)

```bash
curl http://<node-ip>:<nodePort>
```
