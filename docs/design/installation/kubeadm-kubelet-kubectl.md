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
```

Install the latest package versions available in the configured repository:

```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

To install a specific patch version instead, first check which package versions are available:

```bash
# Check which package versions are available in the configured repository
apt-cache madison kubelet
```

For a direct one-line install, replace the example version with the one you selected:

```bash
# Install directly with a specific package version (example: 1.37.1-1.1)
sudo apt-get install -y kubelet=1.37.1-1.1 kubeadm=1.37.1-1.1 kubectl=1.37.1-1.1
```

Alternatively, use a variable to avoid repeating the version:

```bash
K8S_PACKAGE_VERSION=1.37.1-1.1
sudo apt-get install -y \
  kubelet="$K8S_PACKAGE_VERSION" \
  kubeadm="$K8S_PACKAGE_VERSION" \
  kubectl="$K8S_PACKAGE_VERSION"
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

The repository URL selects a minor release (`v1.37`); the package version selects the patch release. `1.37.1-1.1` is only an example: choose the exact version shown by `apt-cache madison kubelet` and set `K8S_PACKAGE_VERSION` to it. For another Kubernetes version, configure the repository URL above to the corresponding minor release and choose an available package version from that repository.

The kubelet restarts in a loop until `kubeadm init` or `kubeadm join` runs. This is expected.

See the official Kubernetes instructions for [installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

## Next

Continue with the setup you chose:

- Single master: [Initialize the master](single-master-init.md)
- Multiple masters: [Set up the load balancer](multi-master-load-balancer.md)
