# Worker Plane Failure: API Server Endpoint

If a worker's kubelet cannot reach the API server, the node may become `NotReady` or `Unknown`.

## Check the kubelet endpoint

On the control-plane node, find its IP and confirm the API server's listening port:

```bash
hostname -I # Shows the control-plane node's IP address(es)
ss -lntp # Shows listening TCP ports and their processes
```

The API server normally listens on TCP port `6443`.

On the worker, inspect the `server` value in the kubelet kubeconfig:

```bash
cat /etc/kubernetes/kubelet.conf
```

Check that the IP address and port in `server: https://<IP>:6443` are correct for the control-plane API server or, in an HA cluster, the load balancer. If either is wrong, edit the file:

```bash
vi /etc/kubernetes/kubelet.conf
```

Then restart the kubelet and verify the node status:

```bash
systemctl restart kubelet
kubectl get nodes
```
