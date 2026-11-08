# Initialize the Master (Single Master)

Run these steps on the master node only.

## Run kubeadm init

Set `--pod-network-cidr` to the range your pod network add-on expects. The value below is the default for Calico.

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

## Set up kubectl

Set up `kubectl` for your user with the commands printed at the end of `kubeadm init`:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

If you are working as `root`, run this instead:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## Next

Continue with [Install the CNI add-on](cni.md).
