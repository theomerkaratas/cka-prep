# Application Failure: Overview (CKA)

Application failures are a large part of the Troubleshooting domain (30%). The exam gives you a broken app in a namespace and asks you to find and fix it. Work the request path from the outside in, and check every link until you find the broken one.

---

## The Request Path

Take a simple two-tier app: a **web** Deployment (pods) behind a **web-svc** Service, which talks to a **db** Deployment behind a **db-svc** Service.

```
User → NodePort/ClusterIP (web-svc) → Endpoints → web Pod → db-svc → Endpoints → db Pod
```

Each arrow is a place where things break. Check them in order:

| # | Check | Command | Typical failure |
| :-: | :--- | :--- | :--- |
| 1 | Can you reach the app? | `curl http://<node-ip>:<nodePort>` or `kubectl port-forward svc/web-svc 8080:80 -n <ns>` | Wrong port, NodePort, or firewall |
| 2 | Does the Service have endpoints? | `kubectl get endpoints web-svc -n <ns>` | Selector mismatch, or no Ready pods |
| 3 | Is the pod Running and Ready? | `kubectl get pods -n <ns> -o wide` and `kubectl describe pod <pod> -n <ns>` | CrashLoopBackOff, ImagePullBackOff, Pending |
| 4 | What do the logs and events say? | `kubectl logs <pod> -n <ns> --previous` and `kubectl get events -n <ns> --sort-by=.lastTimestamp` | Bad config, missing Secret/ConfigMap, app error |
| 5 | Repeat 2–4 for the DB tier | `kubectl get endpoints db-svc -n <ns>`, then the same checks on the DB pod | Same causes; DB pod not Ready |

Start from either end. If you know the symptom is "cannot reach the app", start at the Service. If you know a pod is crashing, start at the pod.

Next: [Triage Commands](triage-commands.md), [Common Failures](common-failures.md), [Exam Workflow](exam-workflow.md).
