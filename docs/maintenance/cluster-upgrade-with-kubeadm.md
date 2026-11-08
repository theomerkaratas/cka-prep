# Cluster Upgrade with kubeadm

Official guide: [Upgrade a kubeadm cluster](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).

## Checking Versions

Run `kubectl get nodes` to see the version of each node:

```text
$ kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   14m   v1.33.0
node01         Ready    <none>          13m   v1.33.0
```

## Version Skew Rules

- `etcd` and `CoreDNS` have their own versions. The Kubernetes components share one version, but they do not all have to run the same version.
- All components talk to the API server, so none of them should run a version higher than the kube-apiserver.
- The recommended order is control plane (master) nodes first, then worker nodes.

## Upgrade Options

- **Managed Kubernetes (cloud):** the provider usually offers an upgrade option directly. For example, on GCP:
  _(Screenshot: `Screenshot 2026-10-22 at 9.22.37 PM.png`, GCP upgrade option)_
- **kubeadm:** the cluster has its own upgrade commands, shown below.
- **Hard way:** if the cluster was set up manually, every component has to be upgraded by hand.

## Upgrading with kubeadm

### Master (Control Plane) Nodes

```bash
# Check which package versions are currently visible to apt:
apt-cache madison kubeadm

# If the target minor version is not listed, update the repository source list file:
# Example: updating from v1.33 to v1.34 in /etc/apt/sources.list.d/
sudo sed -i 's/v1.33/v1.34/g' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
# or just below
vi /etc/apt/sources.list.d/kubernetes.list

# Verify available versions again to pick the exact version string (e.g., 1.34.0-1.1):
apt-cache madison kubeadm

# 1. Unhold kubeadm package, install target version, and hold again
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.34.0-1.1
sudo apt-mark hold kubeadm

# 2. Verify the upgrade plan and component versions
sudo kubeadm upgrade plan

# 3. Apply the upgrade
sudo kubeadm upgrade apply v1.34.0
# Note: This updates the static pods: `kube-apiserver`, `kube-controller-manager`,
# `kube-scheduler`, and `etcd`. `kubelet` is NOT updated by this command.

# 4. Drain the node
kubectl drain <node-name> --ignore-daemonsets

# 5. Upgrade kubelet and kubectl, then hold them
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet=1.34.0-1.1 kubectl=1.34.0-1.1
sudo apt-mark hold kubelet kubectl

# 6. Reload systemd daemon and restart the kubelet service
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 7. Uncordon the control plane node
kubectl uncordon <node-name>
```

### Worker Nodes

```bash
# Drain the worker node
kubectl drain node-name --ignore-daemonsets --delete-emptydir-data

# Unhold package, install new version, and hold again
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.31.0-1.1
sudo apt-mark hold kubeadm

# Upgrade local node configuration
sudo kubeadm upgrade node
# On worker nodes, run `upgrade node` instead of `upgrade apply`. This updates the local kubelet configuration.

# Upgrade `kubelet` and `kubectl`, then restart `kubelet`
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet=1.31.0-1.1 kubectl=1.31.0-1.1
sudo apt-mark hold kubelet kubectl

# Reload systemd daemon and restart the kubelet service
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon the worker node
kubectl uncordon node-name
```

## Example: `kubeadm upgrade plan` Output

Output from an older release, kept as a reference:

```text
$ kubeadm upgrade plan
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade/config] Making sure the configuration is correct:
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.11.8
[upgrade/versions] kubeadm version: v1.11.3
[upgrade/versions] Latest stable version: v1.13.4
[upgrade/versions] Latest version in the v1.11 series: v1.11.8

Components that must be upgraded manually after you have
upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     3 x v1.11.3   v1.13.4

Upgrade to the latest stable version:

COMPONENT            CURRENT    AVAILABLE
API Server           v1.11.8    v1.13.4
Controller Manager   v1.11.8    v1.13.4
Scheduler            v1.11.8    v1.13.4
Kube Proxy           v1.11.8    v1.13.4
CoreDNS              1.1.3      1.1.3
Etcd                 3.2.18     N/A

You can now apply the upgrade by executing the following command:
    kubeadm upgrade apply v1.13.4

Note: Before you can perform this upgrade, you have to update kubeadm to v1.13.4.
```

## Further Reading
- [`kubeadm upgrade` command reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/)
- [Upgrade a kubeadm cluster](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
