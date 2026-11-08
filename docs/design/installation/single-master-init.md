# Initialize the Master (Single Master)

Run these steps on the master node only.

## Run kubeadm init

Set `--pod-network-cidr` to the range your pod network add-on expects. The value below is the default for Calico.

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

`kubeadm` selects an address for the API server from the node's network interfaces. If the node has multiple interfaces or you need to choose a specific address, set `--apiserver-advertise-address`:

Find the node's IP addresses:

```bash
ip addr
hostname -I
```

`ip addr` shows addresses grouped by network interface; `hostname -I` prints the node's IP addresses as a short list. Choose the address on the interface reachable by the other cluster nodes, not a loopback, container, or pod-network address.

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<CONTROL_PLANE_IP> \
  --pod-network-cidr=192.168.0.0/16
```

`--upload-certs` is not needed for a single-master cluster. It uploads control-plane certificates so additional control-plane nodes can retrieve them when joining an HA cluster.

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
