# Kube API Server

The `kube-apiserver` is the central management hub of the Kubernetes control plane. It exposes the Kubernetes API (port `6443`), orchestrates all operations, authenticates and authorizes requests, and is the **only component that directly communicates with `etcd`**.

---

## Deployment Architectures

How `kube-apiserver` runs depends on how the cluster was provisioned:

### 1. `kubeadm` Setup (Static Pod)
When set up using `kubeadm`, the API server runs as a static pod in the `kube-system` namespace on the control plane node.

```text
$ kubectl get pods -n kube-system
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   coredns-78fcdf6894-hwrq9          1/1     Running   0          16m
kube-system   coredns-78fcdf6894-rzhjr          1/1     Running   0          16m
kube-system   etcd-master                       1/1     Running   0          15m
kube-system   kube-apiserver-master             1/1     Running   0          15m
kube-system   kube-controller-manager-master    1/1     Running   0          15m
kube-system   kube-proxy-lzt6f                  1/1     Running   0          16m
kube-system   kube-proxy-zm5qd                  1/1     Running   0          16m
kube-system   kube-scheduler-master             1/1     Running   0          15m
kube-system   weave-net-29z42                   2/2     Running   1          16m
kube-system   weave-net-snmdl                   2/2     Running   1          16m
```

Its configuration is stored in the static pod manifest located at `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --advertise-address=172.17.0.32
    - --allow-privileged=true
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --disable-admission-plugins=PersistentVolumeLabel
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

---

### 2. Manual Setup (Systemd Service)
In a non-`kubeadm` setup (installed manually or from binaries), `kube-apiserver` runs as a systemd service. We can inspect its configuration via `/etc/systemd/system/kube-apiserver.service`:

```ini
# /etc/systemd/system/kube-apiserver.service
[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=${INTERNAL_IP} \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --enable-swagger-ui=true \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \
  --event-ttl=1h \
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \
  --kubelet-https=true \
  --runtime-config=api/all \
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2
```

You can also inspect the running process directly:
```bash
ps aux | grep kube-apiserver
```

