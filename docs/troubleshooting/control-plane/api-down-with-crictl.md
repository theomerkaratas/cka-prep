# Control Plane Failure: When the API Server Is Down

`kubectl` cannot reach the API, so use the container runtime directly:

```bash
crictl ps -a | grep kube-apiserver
crictl logs <container-id>
```

## Check the static pod manifests

```bash
ls /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

The kubelet watches this directory, so saving a fixed manifest restarts the pod automatically. Confirm the path in the kubelet config (`staticPodPath` in `/var/lib/kubelet/config.yaml`).
