# Set Up the Load Balancer (Multiple Masters)

This guide uses three master nodes and one worker. Replace the example addresses with your own.

| Role | Hostname | IP |
| :--- | :--- | :--- |
| Load balancer | `lb` | `<LB_IP>` |
| Control plane | `master1` | `<MASTER1_IP>` |
| Control plane | `master2` | `<MASTER2_IP>` |
| Control plane | `master3` | `<MASTER3_IP>` |
| Worker | `worker1` | `<WORKER1_IP>` |

Three masters are the usual minimum for HA. etcd needs a majority (quorum) to work, so with three members the cluster survives the loss of one.

The official kubeadm HA guide lists three or more worker nodes as a prerequisite. One worker is enough for practice, but add more for real workloads.

## Why a load balancer

The load balancer is the single address that `kubectl`, the kubelets, and the workers use to reach the API servers. It forwards TCP port `6443` to all three masters. Set it up before you run `kubeadm init`.

## Example HAProxy configuration

On the `lb` node:

```text
frontend kubernetes-api
    bind *:6443
    mode tcp
    default_backend kubernetes-masters

backend kubernetes-masters
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 <MASTER1_IP>:6443 check
    server master2 <MASTER2_IP>:6443 check
    server master3 <MASTER3_IP>:6443 check
```

Start HAProxy and check that it listens on port `6443`:

```bash
sudo systemctl enable --now haproxy
sudo ss -tlnp | grep 6443
```

The API servers do not exist yet, so the backends show as down until `kubeadm init` runs on `master1`. This is expected.

> A single load balancer is still a single point of failure. In production, run two load balancers with a virtual IP (for example with keepalived), or use a cloud load balancer.

## Next

Continue with [Initialize the first master](multi-master-init.md).
