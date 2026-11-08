# Prerequisites on All Nodes

Run these steps on every master and worker node before you install anything else.

## Disable swap

The kubelet requires swap to be off. Turn it off now and remove it from `/etc/fstab` so it stays off after a reboot:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## Load the kernel modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

## Set the networking parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Next

Continue with [Container runtime](container-runtime.md).
