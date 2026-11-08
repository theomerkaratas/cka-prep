# Container Runtime (containerd)

Install containerd on all nodes. kubelet talks to containerd through the CRI, so every node needs a running runtime before `kubeadm init` or `kubeadm join`.

```bash
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Use the systemd cgroup driver, which kubelet expects
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Next

Continue with [kubeadm, kubelet, and kubectl](kubeadm-kubelet-kubectl.md).
