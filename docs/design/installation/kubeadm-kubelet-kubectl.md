# Install kubeadm, kubelet, and kubectl

Install these tools on all nodes. The commands below use the Kubernetes v1.37 package repository. Replace `v1.37` with the version you need.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

The kubelet restarts in a loop until `kubeadm init` or `kubeadm join` runs. This is expected.

## Next

Continue with the setup you chose:

- Single master: [Initialize the master](single-master-init.md)
- Multiple masters: [Set up the load balancer](multi-master-load-balancer.md)
