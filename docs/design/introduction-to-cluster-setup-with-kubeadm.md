# Cluster (Single Master) Setup with kubeadm

kubeadm bootstraps a Kubernetes cluster. It installs and configures the control plane components on the master node, and it joins worker nodes to the cluster. Before running kubeadm, each node needs a container runtime and some system settings.

## 1. Prerequisites on all nodes

**Disable swap.** The kubelet requires swap to be off. Turn it off now and remove it from `/etc/fstab` so it stays off after a reboot:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

**Load the kernel modules and set the networking parameters:**

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## 2. Install the container runtime (containerd) on all nodes

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

## 3. Install kubeadm, kubelet, and kubectl on all nodes

The commands below use the Kubernetes v1.31 package repository as an example. Replace `v1.31` with the version you need.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

The kubelet restarts in a loop until `kubeadm init` or `kubeadm join` runs. This is expected.

## 4. Initialize the control plane on the master node

`kubeadm init` installs and configures the control plane components on the master node. Set `--pod-network-cidr` to the range your pod network add-on expects. The value below is the default for Calico.

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Set up `kubectl` for your user with the command printed at the end of `kubeadm init`:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 5. Install the pod network add-on (CNI)

A normal network connection between the nodes is not enough. Kubernetes needs a pod network, and you install it with a CNI add-on after `kubeadm init`. Until the add-on is installed, the nodes stay `NotReady`. Apply the manifest for the add-on you chose:

```bash
kubectl apply -f <CNI_MANIFEST_URL_OR_FILE>
```

## 6. Join the worker nodes

On the master node, print a join command with a new token:

```bash
kubeadm token create --print-join-command
```

Run the printed command with `sudo` on each worker node. It has this form:

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## 7. Verify the cluster

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

All nodes should be `Ready`, and the control plane and add-on pods should be `Running`.

## HA cluster setup: additional steps

An HA cluster needs the steps above, plus the following changes:

1. **Create a load balancer in front of the API servers.** Forward TCP port `6443` to all control plane nodes (for example with NGINX or HAProxy). Do this before `kubeadm init`.

2. **Initialize the first control plane node with the load balancer endpoint and uploaded certificates:**

   ```bash
   sudo kubeadm init \
     --control-plane-endpoint "<LB_IP_OR_DNS>:6443" \
     --upload-certs \
     --pod-network-cidr=192.168.0.0/16
   ```

   `--control-plane-endpoint` makes the cluster address the load balancer instead of a single master. `--upload-certs` stores the control plane certificates in the cluster, so other masters can download them.

   The command prints a `--certificate-key` value. Save it for the next step.

3. **Install the CNI add-on** on the first control plane node, as in step 5 above.

4. **Join the other control plane nodes** with the `--control-plane` flag and the certificate key:

   ```bash
   sudo kubeadm join <LB_IP_OR_DNS>:6443 \
     --token <token> \
     --discovery-token-ca-cert-hash sha256:<hash> \
     --control-plane \
     --certificate-key <certificate-key>
   ```

   The certificate key expires after 2 hours. If it has expired, create a new one on an existing control plane node:

   ```bash
   sudo kubeadm init phase upload-certs --upload-certs
   ```

   Use the new key in the join command.

5. **Join the worker nodes** through the load balancer, using the same join command as in step 6 of the single-master setup but with the load balancer address.

6. **Verify the HA setup:**

   ```bash
   kubectl get nodes
   kubectl -n kube-system get pods -o wide
   ```

   Each control plane node should appear, and each control plane component should run on each master.
