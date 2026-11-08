# Configuring HA

A highly available (HA) cluster has redundancy in its control plane so that no single component or node is a single point of failure. In this section we focus on the control plane: the master nodes and the components they run.

The control plane components:

- API server
- Controller manager
- Scheduler
- ETCD

When you add a second master node, the same components run on it as well. Each component type behaves differently when it runs in multiple instances.

## API server: active-active

The API server receives requests and processes them, and each request is handled by one instance. All API servers can run at the same time in an active-active mode.

`kubectl` talks to the API server on port `6443`, and this address is set in the kubeconfig file. With multiple masters, you should not point `kubectl` at a single master. Instead, place a load balancer (for example NGINX or HAProxy) in front of the master nodes, distribute traffic across the API servers, and point `kubeconfig` at the load balancer address.

## Controller manager and scheduler: active-standby

The controller manager and the scheduler watch cluster state and take actions, such as creating a replacement pod when one fails. If several instances acted at the same time, they could duplicate actions, for example by creating more pods than needed. For this reason, these components run in an active-standby mode, and only one instance acts as the leader at a time.

Leader election is controlled by the `--leader-elect` option, which is `true` by default. The instances compete for a lease on an endpoint object (for example `kube-controller-manager` for the controller manager). The instance that holds the lease is active, and the others stay passive. The active instance renews the lease periodically. If it fails, a passive instance acquires the lease and becomes the leader. The scheduler uses the same mechanism and the same command line options.

## etcd topologies

There are two ways to deploy etcd in an HA cluster:

- **Stacked control plane nodes:** etcd runs on the same nodes as the control plane components. This is easier to set up and manage and needs fewer nodes. However, if one node goes down, both an etcd member and a control plane instance are lost, which reduces redundancy.
- **External etcd cluster:** etcd runs on its own set of servers, separate from the control plane nodes. A failed control plane node does not affect the etcd cluster or the data it stores. This is less risky, but harder to set up and requires twice as many servers.

Regardless of the topology, the API server is the only component that talks to etcd directly. The API server's configuration specifies the etcd endpoints with the `--etcd-servers` option. etcd is a distributed system, so you can read and write data through any of its members. This is why `--etcd-servers` takes a list of endpoints.

## Useful commands

Check the control plane pods and the node each one runs on:

```bash
kubectl -n kube-system get pods -o wide
```

Check which instance currently holds the leader lease for the controller manager and the scheduler. The `HOLDER` column shows the active instance:

```bash
kubectl -n kube-system get lease kube-controller-manager
kubectl -n kube-system get lease kube-scheduler
```

Check the etcd endpoints the API server uses. On kubeadm clusters, the API server runs as a static pod defined in `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```bash
grep etcd-servers /etc/kubernetes/manifests/kube-apiserver.yaml
```

Check the members and health of the etcd cluster. The certificate paths below are the kubeadm defaults:

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster --write-out=table
```

On the exam, the paths or endpoints may differ. Check the static pod manifests to confirm them before running the commands.
