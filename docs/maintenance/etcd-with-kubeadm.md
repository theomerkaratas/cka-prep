# ETCD with kubeadm

Official guide: [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/).

## Overview

- In a `kubeadm` cluster, `etcd` is not a system service. It runs as a static pod managed by `kubelet`.
- To see its configuration:

```bash
cat /etc/kubernetes/manifests/etcd.yaml
```

```yaml
  - etcd
    - --advertise-client-urls=https://127.0.0.1:2379
    - --key-file=/path-to-certs/etcdserver.key
    - --cert-file=/path-to-certs/etcdserver.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://127.0.0.1:2380
    - --initial-cluster=master=https://127.0.0.1:2380
    - --listen-client-urls=https://127.0.0.1:2379
    - --listen-peer-urls=https://127.0.0.1:2380
    - --name=master
    - --peer-cert-file=/path-to-certs/etcdpeer1.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

## Finding the Static Pod Directory

If you don't know where the static pod files are, run:

```bash
ps aux | grep kubelet
ps aux | grep kubelet | grep -i pod-manifest-path
# or
grep -i "staticPodPath" /var/lib/kubelet/config.yaml
```

> The usual path is **`/etc/kubernetes/manifests`**.

The `--data-dir` flag is defined in this file.

## Taking a Snapshot

```bash
# Ensure ETCDCTL uses v3 API
export ETCDCTL_API=3

# Take snapshot
etcdctl snapshot save /opt/snapshot-pre-boot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the backup file
ls -lh /opt/snapshot-pre-boot.db

# View the status of the backup
etcdctl snapshot status /opt/snapshot-pre-boot.db --write-out=table
```

## Restoring a Snapshot

### 1. Stop the API server

```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 30
```

### 2. Restore the snapshot into a new data directory

```bash
etcdctl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir=/var/lib/etcd-from-backup
```

### 3. Point the etcd static pod at the new data directory

```bash
sudo vi /etc/kubernetes/manifests/etcd.yaml
# under spec.volumes.[name: etcd-data].hostPath.path
# or just try to find etcd-data
```

### 4. Restart the API server and check etcd

```bash
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Check whether etcd pod restarted
kubectl get pods -n kube-system -l component=etcd
```

### 5. Restart the remaining control plane components

```bash
# Restart kube-controller-manager
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
sleep 20
mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/

# Restart kube-scheduler
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
sleep 20
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/

# Restart kubelet
systemctl restart kubelet

# Monitor the restart process
watch crictl ps
```

> **REMEMBER:** at the beginning, execute `export ETCDCTL_API=3`.
