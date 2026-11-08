# Installation with kubeadm

kubeadm bootstraps a Kubernetes cluster. It installs and configures the control plane components on the master nodes, and it joins worker nodes to the cluster. The steps are the same for both setups below, except for the control plane part.

## Choose a setup

| Setup | Control plane | Use it for |
| :--- | :--- | :--- |
| **Single master** | One master node | Learning, labs, and quick test clusters. |
| **Multiple masters (HA)** | Three master nodes behind a load balancer | Clusters that must keep working when a master fails. |

## Steps

Follow the steps in order. The shared steps are used by both setups.

**Shared steps (all nodes)**

1. [Prerequisites](prerequisites.md): disable swap, kernel modules, and sysctl settings.
2. [Container runtime](container-runtime.md): install containerd.
3. [kubeadm, kubelet, and kubectl](kubeadm-kubelet-kubectl.md): install the Kubernetes tools.

**Single master**

4. [Initialize the master](single-master-init.md)
5. [Install the CNI add-on](cni.md)
6. [Join the worker nodes](join-workers.md)
7. [Verify the cluster](single-master-verify.md)

**Multiple masters**

4. [Set up the load balancer](multi-master-load-balancer.md)
5. [Initialize the first master](multi-master-init.md)
6. [Install the CNI add-on](cni.md)
7. [Join the other masters](multi-master-join-control-planes.md)
8. [Join the worker nodes](join-workers.md)
9. [Verify the HA cluster](multi-master-verify.md)

## Further Reading

- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
