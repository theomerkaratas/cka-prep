# DaemonSets

A **DaemonSet** ensures that all (or a subset of) nodes run a copy of a Pod. When new nodes are added to the cluster, Pods are automatically scheduled onto them. When nodes are removed, the Pods are garbage collected.

---

## Common Use Cases

- **Cluster Storage Daemons:** `ceph`, `glusterd`
- **Log Collection Daemons:** `fluentd`, `logstash`, `promtail`
- **Node Monitoring Daemons:** `prometheus-node-exporter`, `datadog-agent`
- **Cluster Networking Addons:** `kube-proxy`, `calico-node`, `weave-net`

---

## DaemonSet Manifest

The structure of a DaemonSet manifest is very similar to a ReplicaSet or Deployment, but it has no `replicas` field:

```yaml
# daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # These tolerations ensure daemonset runs on control plane / master nodes as well
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
```

---

## CKA Exam Tip: Generating DaemonSet YAML

There is **no direct imperative command** like `kubectl create daemonset`. The fastest way to generate a DaemonSet manifest during the exam is:

```bash
# 1. Generate a deployment template
kubectl create deployment my-daemon --image=fluentd --dry-run=client -o yaml > daemonset.yaml

# 2. Modify the file:
#    - Change 'kind: Deployment' to 'kind: DaemonSet'
#    - Delete the 'replicas' line
#    - Delete the 'strategy' section (if present)
#    - Verify 'apiVersion: apps/v1'
```

---

## Inspecting DaemonSets

```bash
# List all DaemonSets in a namespace
kubectl get daemonsets -n kube-system
# Shorthand:
kubectl get ds -n kube-system

# Inspect DaemonSet details
kubectl describe ds fluentd-elasticsearch -n kube-system
```
