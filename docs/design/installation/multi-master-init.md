# Initialize the First Master (Multiple Masters)

Run these steps on `master1` only.

## Run kubeadm init

Use the load balancer address as the control plane endpoint. The `--upload-certs` flag stores the control plane certificates in the cluster, so the other masters can download them:

```bash
sudo kubeadm init \
  --control-plane-endpoint "<LB_IP_OR_DNS>:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16
```

- `--control-plane-endpoint` makes the cluster use the load balancer address instead of `master1`. Turning a cluster created without this flag into an HA cluster is not supported by kubeadm, so set it from the start.
- `--upload-certs` lets the other masters join without copying certificate files by hand.

## Save the join values

The output prints two values that you need in the next steps:

- a `kubeadm join ... --control-plane` command with `--token` and `--discovery-token-ca-cert-hash`
- a `--certificate-key` value

Save both. The certificate key gives access to cluster secrets, so keep it private.

## Set up kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

As `root`, use `export KUBECONFIG=/etc/kubernetes/admin.conf` instead.

## Next

Install the CNI add-on on `master1`: [Install the CNI add-on](cni.md). Then continue with [Join the other masters](multi-master-join-control-planes.md).
