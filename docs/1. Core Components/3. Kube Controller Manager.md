# Kube Controller Manager

The `kube-controller-manager` is a control plane daemon that embeds the core control loops shipped with Kubernetes.

- **What is a Controller?** A continuous control loop that watches the shared state of the cluster through the `kube-apiserver` and makes changes attempting to move the current state towards the desired state.
- **Examples of Controllers:**
  - **Node Controller:** Responsible for noticing and responding when nodes go down (e.g. `--node-monitor-period=5s`, `--node-monitor-grace-period=40s`, `--pod-eviction-timeout=5m0s`).
  - **Replication Controller / ReplicaSet Controller:** Ensures the correct number of pods are running.
  - **Deployment Controller:** Manages rollout and updates of Deployments.
  - **Namespace Controller:** Cleans up resources when a namespace is deleted.
  - **ServiceAccount Controller:** Creates default accounts and API access tokens for new namespaces.

---

## Installing kube-controller-manager (Manual Setup)

If you are setting up the control plane manually from scratch (without `kubeadm`):

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.30.0/bin/linux/amd64/kube-controller-manager
```

Sample systemd service definition (`/etc/systemd/system/kube-controller-manager.service`):

```ini
[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=0.0.0.0 \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --leader-elect=true \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --use-service-account-credentials=true \
  --node-monitor-period=5s \
  --node-monitor-grace-period=40s \
  --pod-eviction-timeout=5m0s \
  --v=2
Restart=on-failure
RestartSec=5
```

---

## Enabling / Disabling Specific Controllers

You can customize which controllers are enabled via the `--controllers` parameter:

```text
--controllers stringSlice     Default: [*]
A list of controllers to enable. '*' enables all on-by-default controllers, 
'foo' enables the controller named 'foo', '-foo' disables the controller named 'foo'.

All controllers: attachdetach, bootstrapsigner, clusterrole-aggregation, cronjob, 
csrapproving, csrcleaner, csrsigning, daemonset, deployment, disruption, 
endpoint, garbagecollector, horizontalpodautoscaling, job, namespace, nodeipam, 
nodelifecycle, persistentvolume-binder, persistentvolume-expander, podgc, 
pv-protection, pvc-protection, replicaset, replicationcontroller, resourcequota, 
root-ca-cert-publisher, route, service, serviceaccount, serviceaccount-token, 
statefulset, tokencleaner, ttl, ttl-after-finished

Disabled-by-default controllers: bootstrapsigner, tokencleaner
```

---

## `kubeadm` Setup (Static Pod)

In a cluster provisioned with `kubeadm`, the controller manager runs as a static pod in `kube-system`:

```bash
kubectl get pods -n kube-system -l component=kube-controller-manager
```

Its manifest file is located at `/etc/kubernetes/manifests/kube-controller-manager.yaml`:

```yaml
# /etc/kubernetes/manifests/kube-controller-manager.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --address=127.0.0.1
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --use-service-account-credentials=true
```

---

## Inspecting the Running Process

To inspect the running controller manager process and its effective CLI arguments:

```bash
ps aux | grep kube-controller-manager
```


