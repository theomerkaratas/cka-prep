# Install the Pod Network Add-on (CNI)

A normal network connection between the nodes is not enough. Kubernetes needs a pod network, and you install it with a CNI add-on after `kubeadm init`. Until the add-on is installed, the nodes stay `NotReady`, and CoreDNS does not start.

Apply the manifest for the add-on you chose. Run this on the first control plane node:

```bash
kubectl apply -f <CNI_MANIFEST_URL_OR_FILE>
```

Use only one pod network add-on per cluster. Set `--pod-network-cidr` in `kubeadm init` to the value your add-on expects. The examples in this section use `192.168.0.0/16`, the default for Calico.

## Next

Continue with:

- Single master: [Join the worker nodes](join-workers.md)
- Multiple masters: [Join the other masters](multi-master-join-control-planes.md)
