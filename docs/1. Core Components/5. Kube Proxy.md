# Kube Proxy

`kube-proxy` is a network proxy that runs on each node in the Kubernetes cluster. It implements part of the Kubernetes Service concept.

---

## What Does Kube-Proxy Do?

- In Kubernetes, every Pod gets its own unique IP address. However, Pods are ephemeral and their IPs can change.
- Services provide a stable virtual IP (`ClusterIP`) and DNS name to reach a group of Pods.
- `kube-proxy` continuously watches the `kube-apiserver` for the creation, modification, or deletion of **Services** and **Endpoints / EndpointSlices**.
- Whenever a Service is created or changed, `kube-proxy` programs routing rules on the local node (using Linux packet filtering mechanisms) to translate the Service's virtual IP into one of the actual backend Pod IPs.

---

## Proxy Modes

`kube-proxy` can operate in different modes:

| Mode | Description |
| :--- | :--- |
| **`iptables`** | **Default mode.** Uses Netfilter iptables rules to capture traffic destined for a Service ClusterIP and redirect it randomly to a backend Pod. |
| **`ipvs`** | Uses Linux IP Virtual Server. Provides higher throughput, faster rule sync for large clusters (>10,000 services), and supports advanced load-balancing algorithms (round-robin, least connections, etc.). |
| **`nftables`** | Modern replacement for iptables, available in newer Kubernetes versions. |

---

## Deployment as a DaemonSet

Because `kube-proxy` must run on every node to configure local routing rules, it is deployed as a **DaemonSet** in the `kube-system` namespace.

```bash
# Check the DaemonSet
kubectl get daemonset kube-proxy -n kube-system

# Check the running kube-proxy pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

---

## Inspecting kube-proxy Configuration

`kube-proxy` reads its settings from a `ConfigMap` in the `kube-system` namespace:

```bash
# View the kube-proxy configuration
kubectl get configmap kube-proxy -n kube-system -o yaml
```

In the ConfigMap, you can check or modify parameters such as:
- `mode`: `"iptables"` or `"ipvs"`
- `clusterCIDR`: The Pod network CIDR range
- `bindAddress`: IP address for the proxy server to bind to (e.g. `0.0.0.0`)

To view logs for troubleshooting:
```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy
```
