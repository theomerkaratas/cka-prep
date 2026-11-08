# Application Failure: Exam Workflow and Tips

## Exam Workflow (Step by Step)

1. **Set the namespace:** `kubectl config set-context --current --namespace=<ns>` (or use `-n` on every command).
2. **Get the overview:** `kubectl get all` and `kubectl get pods -o wide`.
3. **Test the entry point:** curl the NodePort or `port-forward` to the Service.
4. **Check the Service endpoints:** `kubectl get endpoints <svc>`. If empty, compare selectors and labels.
5. **Check the pods:** status, restarts, and `describe` events.
6. **Check the logs:** current, then `--previous` if restarting.
7. **Repeat for the DB tier.**
8. **Fix, then re-test** from the entry point. Do not move on until the app responds.

---

## Exam Tips

- **Read the Events first.** `kubectl describe` is the fastest way to see the root cause.
- **Use `--previous` for crashing pods.** Current logs often show nothing useful.
- **Check labels and selectors character by character.** A single typo breaks the Service.
- **Check `targetPort` against the container port.** The Service `port` is what clients use; `targetPort` is where traffic goes.
- **Use `kubectl explain`** to confirm field names instead of searching.
- **Do not forget the DB tier.** Once the frontend works, check whether the DB side is fine too.

### References
- [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
