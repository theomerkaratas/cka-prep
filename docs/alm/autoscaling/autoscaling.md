Scaling infra -> scaling nodes
Scaling workloads -> scaling pods

we can do it manually and automatically

## Manual Way
- we can add new node with `kubeadm join`
- we can manually horizontal scaling pods with `kubectl scale`
- we can manually vertical scaling pods with `kubectl edit`

## Auto Way
- for cluster, we have `Cluster Autoscaler`
- for pods, we have HPA and VPA
