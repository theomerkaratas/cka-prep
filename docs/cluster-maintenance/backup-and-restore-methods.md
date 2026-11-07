- we should backup `etcd`
- if out applications are configured with persistent storage, then they are candidate for backups
- also we should backup resource configurations (like definition yaml files)


## Resource Configuration
- we can backup them with query to `kube-apiserver`
```sh
kubectl get all -A -o yaml > all-deploy-services.yaml
```
> Note: this does not include `ConfigMap`, `Secret`, `Ingress`, `PersistentVolume`, `PersistentVolumeClaim`, `CRD` and `RBAC`

better approach is using `Velero`, for this approach
# ETCD
- store info about state
- instead of backup resource, we may choose backup the `etcd server` itself
- hoster on master nodes
- while configuring it, we specified a location where all the data would be stored with `--data-dir` flag
- data directory is the directory that can be configured to be backed up by our backup tool

```text
$ cat etcd.service
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://${INTERNAL_IP}:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
```

- `etcd` also. comes with a built in snapshot solutions
```sh
# Ensure ETCDCTL uses v3 API
export ETCDCTL_API=3

# Take snapshot (with certificates and endpoint for secured clusters)
etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key

# Verify backup file
ls -lh snapshot.db

# View the status of the backup
etcdctl snapshot status snapshot.db --write-out=table
```

```text
$ etcdctl \
    snapshot status snapshot.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| e63b3fc5 |   473353 |        875 |     4.1 MB |
+----------+----------+------------+------------+
```

```sh
# Restore process (for systemd / Hard Way setup):
# 1. Stop API Server to prevent new writes
systemctl stop kube-apiserver

# 2. Restore snapshot into a NEW data directory
etcdctl snapshot restore --data-dir=/var/lib/etcd-from-backup snapshot.db

# 3. Ensure the etcd service user has permissions to the restored data directory
chown -R etcd:etcd /var/lib/etcd-from-backup

# 4. Update the etcd service definition with the new --data-dir path
systemctl status etcd # locate the loaded service file path (e.g., /etc/systemd/system/etcd.service)
vi /etc/systemd/system/etcd.service

# 5. Reload systemd and restart services
systemctl daemon-reload
systemctl restart etcd
systemctl start kube-apiserver
```

>run the `etcdctl` snapshot restore command with the path set to the path of the backup file, which is the snapshot.db file. when `etcd` restore form a backup, it initializes a new cluster configuration and configure the members of `etcd` as new members of a new cluster. this is the prevent a new member from accidentally joining an existing cluster. when we run the command above, a new data directory is created (in our example, `/var/lib/etcd-from-backup`). We then configure the `etcd` configuration file the use the new data  directory, then reload the service daemon and restart `etcd` service


when we use `etcdctl` commands, we should specify the certificate files, endpoint, ca certificate, `etcd` server certificate and the key  for authentication, like below:
```sh
etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key
```
> so  we have two options; a backup using `etcd` and querying `kube-apiserver`. 
> both of these have pros and cons. 
> if we are using a managed k8s environment, then we may not even have access to the `etcd` cluster. 


> REMEMBER!! In the beginning, execute `export ETCDCTL_API=3`