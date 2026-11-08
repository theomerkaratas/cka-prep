# Control Plane Failure: When the API Server Is Down

`kubectl` cannot reach the API, so use the container runtime directly:

```bash
crictl ps -a | grep kube-apiserver
crictl logs <container-id>
```

## Check the static pod manifests

```bash
ls /etc/kubernetes/manifests/
# etcd.yaml kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml
```

`kube-proxy` is not a static pod, so it does not have a manifest in this directory. It runs as a DaemonSet; edit its configuration through the `kube-proxy` ConfigMap with `kubectl edit configmap kube-proxy -n kube-system`.

Check the manifest for the affected component and compare its settings with that component's logs and events:

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

The kubelet watches this directory, so saving a fixed manifest restarts the corresponding pod automatically. Confirm the path in the kubelet config (`staticPodPath` in `/var/lib/kubelet/config.yaml`).
