# Step 4: Check CoreDNS

CoreDNS turns service names into IP addresses. If it is broken, pods cannot find each other by name.

## What CoreDNS Is Made Of

- CoreDNS runs as **pods** in the `kube-system` namespace, managed by a **Deployment**.
- A **kube-dns Service** exposes these pods on port 53.
- A **ConfigMap** named `coredns` holds the **Corefile**, which sets how CoreDNS behaves.
- A **ServiceAccount**, **ClusterRole**, and **ClusterRoleBinding** give CoreDNS the access it needs.
- Each pod's `/etc/resolv.conf` points to the kube-dns Service IP.

A typical Corefile looks like this:

```text
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
    }
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

- `kubernetes cluster.local` answers names inside the cluster.
- `forward . /etc/resolv.conf` sends all other names to the node's DNS servers.

## Step 4.1: Check the CoreDNS Pods

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

| STATUS | Meaning | Next action |
| :--- | :--- | :--- |
| `Running` | Good | Go to Step 4.2 |
| `Pending` | Not scheduled, often a CNI problem | Go to [Step 6](cni.md) |
| `CrashLoopBackOff` | The container keeps crashing | Read the logs below |

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

## Step 4.2: Check the kube-dns Endpoints

```bash
kubectl get endpoints kube-dns -n kube-system
```

If there are no endpoints, the kube-dns Service is not finding the CoreDNS pods. Check the Service selector and the pod labels:

```bash
kubectl get svc kube-dns -n kube-system -o yaml
kubectl get pods -n kube-system --show-labels
```

## Step 4.3: Check the Corefile

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Look for typos, a missing `kubernetes` block, or a wrong `forward` line.

## What to Do Next

- **CoreDNS pods are `Running` and kube-dns has endpoints:** go to [Step 5](dns-resolution.md).
