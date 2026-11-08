# Install the Pod Network Add-on (CNI)

A normal network connection between the nodes is not enough. Kubernetes needs a pod network, and you install it with a CNI add-on after `kubeadm init`. Until the add-on is installed, the nodes stay `NotReady`, and CoreDNS does not start.

Apply the manifest for the add-on you chose. Run this on the first control plane node:

```bash
kubectl apply -f <CNI_MANIFEST_URL_OR_FILE>
```

Calico is used in some examples, but it is not the only option. Other pod network add-ons, such as Flannel or Cilium, can also be used. Choose one that fits your cluster and follow its installation requirements; use only one pod network add-on per cluster. Set `--pod-network-cidr` in `kubeadm init` to the value your chosen add-on expects.

## Install Flannel

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

After setting up `kubectl`, install Flannel from its latest release manifest on the first control-plane node:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Flannel's default network uses the Pod CIDR `10.244.0.0/16`, so set it when initializing the cluster. If the cluster was initialized with a different CIDR, configure Flannel's manifest to match that network before applying it. See the [official Flannel installation instructions](https://github.com/flannel-io/flannel#deploying-flannel-manually) for details and requirements.

## Next

Continue with:

- Single master: [Join the worker nodes](join-workers.md)
- Multiple masters: [Join the other masters](multi-master-join-control-planes.md)
