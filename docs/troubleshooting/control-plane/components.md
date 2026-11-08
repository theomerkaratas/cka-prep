# Control Plane Failure: Components (CKA)

Control plane failures are part of the Troubleshooting domain (30%). The exam may break `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, or `etcd`, and asks you to find and fix the cause. Check the nodes first, then the components, then their logs.

---

## Components to Know

| Component | Role | If it fails |
| :--- | :--- | :--- |
| `kube-apiserver` | Front door for all requests | `kubectl` stops working |
| `etcd` | Cluster data store | API server cannot read or write state |
| `kube-scheduler` | Assigns pods to nodes | Pods stay `Pending` |
| `kube-controller-manager` | Runs controllers (ReplicaSet, Node, etc.) | Replicas not created, nodes not marked unhealthy |
| `kubelet` (every node) | Runs pods on the node | Node `NotReady` |

With `kubeadm`, the first four run as **static pods** in `kube-system`, defined by manifests in `/etc/kubernetes/manifests/`. Without `kubeadm`, they may run as **systemd services**.

Next: [Triage Flow](triage-flow.md), [Commands](node-and-pod-status.md), [Common Failures](common-failures.md), [Exam Workflow](exam-workflow.md).
