- in `kubeadm` cluster, `etcd` is not system service. it is a static pod run by `kubelet`
- we can see configuration of `etcd` via below:
```sh
cat /etc/kubernetes/manifests/etcd.yaml
```

```yaml
$ cat etcd.yaml
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

if we don't know where are the static pods' files, we should execute the commands below:
```sh
ps aux | grep kubelet
ps aux | grep kubelet | grep -i pod-manifest-path
# or
grep -i "staticPodPath" /var/lib/kubelet/config.yaml
```
> usually it is  **`/etc/kubernetes/manifests`**

in that file we should find `--data-dir`
```sh
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

# Stop kubeapi-server
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 30

# Restore snapshot into a NEW data directory
etcdctl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir=/var/lib/etcd-from-backup
  
# Update etcd's static pod definition file
sudo vi /etc/kubernetes/manifests/etcd.yaml
# under spec.volumes.[name: etcd-data].hostPath.path 
# or just try to find etcd-data

# Restart the kube-apiserver
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Check whether etcd pod restarted
kubectl get pods -n kube-system -l component=etcd
```

```sh
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

> REMEMBER!! In the beginning, execute `export ETCDCTL_API=3`