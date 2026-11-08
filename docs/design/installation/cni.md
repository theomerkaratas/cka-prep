# Install the Pod Network Add-on (CNI)

A normal network connection between the nodes is not enough. Kubernetes needs a pod network, and you install it with a CNI add-on after `kubeadm init`. Until the add-on is installed, the nodes stay `NotReady`, and CoreDNS does not start.

Apply the manifest for the add-on you chose. Run this on the first control plane node:

Download it with a name you choose (`-o`):

```bash
curl -fL -o manifest.yaml "https://raw.githubusercontent.com/OWNER/REPO/BRANCH/PATH/FILE.yaml"
kubectl apply -f manifest.yaml
```

Or let `curl` use the filename from the URL (`-O`, uppercase letter O):

```bash
curl -fLO "https://raw.githubusercontent.com/OWNER/REPO/BRANCH/PATH/FILE.yaml"
kubectl apply -f FILE.yaml
```

Replace the URL placeholders with the GitHub repository, branch, and YAML file path. In both commands, `-f` makes curl fail on HTTP error responses and `-L` follows redirects.

Or skip downloading the file and apply the manifest directly from its URL:

```bash
kubectl apply -f "https://raw.githubusercontent.com/OWNER/REPO/BRANCH/PATH/FILE.yaml"
```

Calico is used in some examples, but it is not the only option. Other pod network add-ons, such as Flannel or Cilium, can also be used. Choose one that fits your cluster and follow its installation requirements; use only one pod network add-on per cluster. Set `--pod-network-cidr` in `kubeadm init` to the value your chosen add-on expects.

## Install Flannel

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

After setting up `kubectl`, download Flannel's latest release manifest on the first control-plane node:

```bash
curl -fL -o kube-flannel.yml https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

In `kube-flannel.yml`, edit the `kube-flannel-cfg` ConfigMap's `net-conf.json` and set `Network` to the same Pod CIDR used with `kubeadm init`:

```yaml
net-conf.json: |
  {
    "Network": "10.244.0.0/16",
    "Backend": {
      "Type": "vxlan"
    }
  }
```

For example, if `kubeadm init` used `--pod-network-cidr=10.244.0.0/16`, set `Network` to `10.244.0.0/16`. Then apply the edited manifest:

```bash
kubectl apply -f kube-flannel.yml
```

The manifest defaults to `10.244.0.0/16`. If the cluster uses a different Pod CIDR, replace that default in the downloaded manifest so it matches `kubeadm init`. The `/32` address shown for `flannel.1` by `ip a` is a node's tunnel address, not the cluster Pod CIDR. See the [official Flannel installation instructions](https://github.com/flannel-io/flannel#deploying-flannel-manually) for details and requirements.

## Next

Continue with:

- Single master: [Join the worker nodes](join-workers.md)
- Multiple masters: [Join the other masters](multi-master-join-control-planes.md)
