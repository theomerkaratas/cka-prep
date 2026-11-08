# Step 5: Check DNS Resolution

Check what a pod sees when it looks up names, and test resolution from a temporary pod.

## Step 5.1: Check the Pod's resolv.conf

Replace `<pod-name>` with a pod that has the DNS problem:

```bash
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
```

Check these lines:

| Line | What it should be |
| :--- | :--- |
| `nameserver` | The ClusterIP of the kube-dns Service |
| `search` | Includes `<namespace>.svc.cluster.local` and `svc.cluster.local` |
| `options ndots:5` | Kubernetes default. Short names are tried with search domains first. |

Get the kube-dns ClusterIP to compare:

```bash
kubectl get svc kube-dns -n kube-system
```

## Step 5.2: Test Resolution from a Temporary Pod

```bash
# Test a built-in service
kubectl run busybox --rm -it --restart=Never --image=busybox:1.28 -- nslookup kubernetes.default.svc.cluster.local

# Test your own service
kubectl run busybox --rm -it --restart=Never --image=busybox:1.28 -- nslookup hostnames.default.svc.cluster.local
```

## What to Do Next

- **`nslookup` works:** DNS is fine. Go to [Step 6](cni.md) only if pods still cannot talk to each other by IP.
- **`nslookup` fails for every name:** DNS is broken. Go back to [Step 4](coredns.md).
- **The kube-dns name works but your service name fails:** check the service name and namespace in the lookup. Then go back to [Step 3](service-endpoints.md).
