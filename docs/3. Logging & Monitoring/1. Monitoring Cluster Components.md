# Monitoring Cluster Components

Kubernetes does not include a full-featured, persistent monitoring solution out-of-the-box. Instead, it provides the **Metrics Server** for real-time, in-memory resource consumption metrics.

---

## Architecture: cAdvisor and Metrics Server

- **cAdvisor (Container Advisor):** An open-source agent built directly into the `kubelet` binary. It collects, aggregates, and exports resource usage data (CPU, memory, filesystem, network) for all containers running on the node.
- **Metrics Server:** An in-memory, cluster-wide aggregator of resource usage data. It periodically queries the `cAdvisor` endpoints exposed by the `kubelet` on each node.
  - **In-Memory Only:** It does not persist historical metrics to disk. To view long-term performance trends or historical metrics, an external solution such as **Prometheus**, **ELK**, or **Datadog** must be deployed.

---

## Installing Metrics Server

You can install Metrics Server by applying the official components manifest:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> **CKA Lab Tip:** In test/lab environments using self-signed certificates, the Metrics Server pod may crash with x509 certificate validation errors. You can resolve this by adding `--kubelet-insecure-tls` to the container arguments in the Metrics Server deployment:
> ```yaml
> spec:
>   containers:
>   - name: metrics-server
>     args:
>     - --kubelet-insecure-tls
> ```

---

## Querying Performance Metrics (`kubectl top`)

Once Metrics Server is healthy and running, you can inspect resource usage using `kubectl top`:

### Node Metrics
```bash
# View all nodes
kubectl top node

# Sample Output:
# NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# controlplane 166m         8%     1337Mi          70%
# node01       36m          1%     1046Mi          55%
```

### Pod Metrics
```bash
# View pods in current namespace
kubectl top pod

# View pods across all namespaces
kubectl top pod -A

# Sort pods by CPU or Memory consumption
kubectl top pod -A --sort-by=cpu
kubectl top pod -A --sort-by=memory

# View resource consumption per container inside pods
kubectl top pod my-pod --containers
```


