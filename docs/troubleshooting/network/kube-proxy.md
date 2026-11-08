# Step 7: Check kube-proxy

kube-proxy runs on every node. It watches Services and Endpoints, then writes the rules (iptables or IPVS) that send service traffic to the right pod.

Use this step when DNS works but you still cannot reach the Service.

## Step 7.1: Check the kube-proxy Pods

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

Every node should have one `kube-proxy` pod in `Running` state.

## Step 7.2: Check the kube-proxy Logs

```bash
kubectl logs -n kube-system <kube-proxy-pod-name>
```

Look for errors about the config file, the proxy mode, or the cluster CIDR.

## Step 7.3: Check the kube-proxy Configuration

`kube-proxy` runs as a DaemonSet, not a static pod, so it has no manifest in `/etc/kubernetes/manifests/`. Its configuration is stored in the `kube-proxy` ConfigMap:

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml
```

Check these settings:

- `mode`: `iptables` or `ipvs`
- `clusterCIDR`: must match the pod network

To edit the configuration:

```bash
kubectl edit configmap kube-proxy -n kube-system
```

The config file is usually mounted inside the kube-proxy pod at `/var/lib/kube-proxy/config.conf`; it may not exist at that path on the node itself.

## Step 7.4: Check the Rules on the Node (Advanced)

Log in to the node first.

```bash
# iptables mode
iptables -t nat -L KUBE-SERVICES -n

# IPVS mode
ipvsadm -Ln
```

Find your Service IP in the output. Each Service should list the pod IPs that it load-balances to. If the Service IP is missing, kube-proxy did not write the rule.

## What to Do Next

- **Rules exist and the pods are healthy:** the network path works. Go back to [Step 2](direct-pod-access.md) to confirm each pod answers.
- **Rules are missing:** check the kube-proxy logs and config. Restart the kube-proxy pod after you fix the cause:

```bash
kubectl delete pod <kube-proxy-pod-name> -n kube-system
```

The DaemonSet creates a new pod automatically.
