- `restartPolicy` applies at the container level or the pod level
- `initContainer` runs before the main containers and shut down after when the main containers complete
- `init-1` -> `init-2` -> `main`
	- if `init-2` fail, `kubelet` will not start `init-1`, because it is already succeeded
	- if pod collapses, `kubelet` rerun all the process

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: myapp
spec:
  initContainers:
  - name: init-myservice
    image: busybox:1.31
    command: ["sh", "-c", "until nslookup my service; do echo waiting for my service; sleep 2; done;"]
  - name: init-mydb
    image: busybox:1.31
    command: ["sh", "-c", "until nslookup mydb; do echo waiting for mydb; sleep 2;done;"]
  containers:
  - name: myapp-container
    image: myimage
    command: ["sh", "-c", "echo The app is running && sleep 3600"]
```

## Further Reading
- [Init Containers documentation](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
