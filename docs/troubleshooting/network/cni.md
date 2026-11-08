# Step 6: Check the CNI Plugin

The CNI plugin gives each pod an IP address and sets up pod-to-pod traffic. Without a working CNI, pods stay `Pending` or cannot reach each other.

## Popular CNI Plugins

| Plugin | Notes |
| :--- | :--- |
| **Flannel** | Simple to set up. Uses an overlay network. Does not support Network Policies by itself. |
| **Calico** | Strong performance and Network Policy support. Works with or without an overlay. |

Install a plugin with the manifest from its official documentation, using `kubectl apply -f <manifest>`.

## Where the CNI Files Are

| What | Where | Notes |
| :--- | :--- | :--- |
| CNI config (`.conf`, `.conflist`) | `/etc/cni/net.d/` | The kubelet reads this folder. |
| CNI binaries (`bridge`, `flannel`, `calico`, `host-local`, ...) | `/opt/cni/bin/` | The plugin executables. |
| CNI manifest (yaml) | Not on the node | Applied with `kubectl apply`. It is stored in the cluster as a DaemonSet. |

Find the DaemonSets that run the plugin:

```bash
kubectl get ds -A
```

Read the manifest that is in the cluster:

```bash
kubectl get ds <ds-name> -n <namespace> -o yaml
```

## Step 6.1: Check the CNI Config Files on the Node

Run this on the node:

```bash
ls -la /etc/cni/net.d/
ls -la /opt/cni/bin/
```

If there is more than one config file, the kubelet uses the **first one in alphabetical order**. Check that this is the one you expect.

If `/etc/cni/net.d/` is empty, the plugin was never installed on this node.

## Step 6.2: Check the CNI Pods

The CNI pods run as a DaemonSet in their own namespace. Check the namespace for your plugin:

```bash
# Flannel
kubectl get pods -n kube-flannel -o wide

# Calico
kubectl get pods -n calico-system -o wide
```

Not sure which namespace? Search all namespaces:

```bash
kubectl get pods -A -o wide | grep -i -E "flannel|calico|weave"
```

## Step 6.3: Check the CNI Pod Logs

```bash
kubectl logs -n kube-flannel <pod-name>
kubectl logs -n calico-system <pod-name>
```

Look for errors that name a file, a subnet, or a permission problem.

## No CNI Pod Is Running

If no CNI pod runs, check in this order:

```bash
# 1. Does the DaemonSet exist, and how many pods does it want?
kubectl get ds -A
kubectl describe ds <ds-name> -n <namespace>

# 2. Why is the pod not running?
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>

# 3. What does the node and kubelet say?
kubectl describe node <node>
journalctl -u kubelet --no-pager | grep -i cni | tail -20
```

| What you see | Likely cause |
| :--- | :--- |
| No DaemonSet in `kubectl get ds -A` | The plugin was never applied. Apply its manifest. |
| DaemonSet exists, `DESIRED` is greater than `CURRENT` | Taint or toleration problem, or a node selector mismatch. |
| Pod `ImagePullBackOff` | Wrong image name or tag, or no registry access. |
| Pod `CrashLoopBackOff` | Read `kubectl logs <pod> -n <namespace> --previous`. |
| Kubelet log says `cni config uninitialized` or `NetworkPluginNotReady` | `/etc/cni/net.d/` is empty or broken. The node stays `NotReady`. |

**Flannel note:** the Flannel pod's `install-cni` init container copies the config to `/etc/cni/net.d/` and the binary to `/opt/cni/bin/`. If the pod never starts, these files are never created.

## What to Do Next

- **CNI pods are `Running` and their logs are clean:** go to [Step 7](kube-proxy.md).
- **CNI pods are not `Running`:** fix the plugin pods first. Then check Step 4 again, because CoreDNS may be `Pending` for the same reason.
