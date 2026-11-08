# Backup and Restore Methods

## What to Back Up

- **`etcd`:** the cluster's state.
- **Applications with persistent storage:** these are candidates for backup.
- **Resource configurations:** the definition YAML files.

## Option 1: Resource Configuration Backup

Query the `kube-apiserver` and save the output:

```bash
kubectl get all -A -o yaml > all-deploy-services.yaml
```

> **Note:** this does not include `ConfigMap`, `Secret`, `Ingress`, `PersistentVolume`, `PersistentVolumeClaim`, `CRD` and `RBAC`.

A better approach is to use **Velero**.

## Option 2: ETCD Backup

- `etcd` stores the cluster's state. Instead of backing up resources, you can back up the `etcd` server itself.
- It runs on the master nodes.
- When configuring it, you specify where all data is stored with the `--data-dir` flag. This is the directory your backup tool should back up.

Example `etcd.service` (hard way):

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

### Taking a Snapshot

`etcd` comes with a built-in snapshot tool:

```bash
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

Example output of the status command:

```text
$ etcdctl \
    snapshot status snapshot.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| e63b3fc5 |   473353 |        875 |     4.1 MB |
+----------+----------+------------+------------+
```

### Restoring a Snapshot (systemd / Hard Way)

```bash
# 1. Stop API server to prevent new writes
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

> When `etcd` restores from a backup, it initializes a new cluster configuration and configures its members as new members of a new cluster. This prevents a new member from accidentally joining an existing cluster. Running the restore command creates a new data directory (in this example, `/var/lib/etcd-from-backup`). Then point the `etcd` configuration at the new data directory, reload the service daemon and restart the `etcd` service.

### Connection Options

When using `etcdctl`, specify the endpoint, the CA certificate, the etcd server certificate and its key for authentication:

```bash
etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key
```

## Choosing an Approach

- You have two options: back up with `etcd`, or query the `kube-apiserver`. Both have pros and cons.
- In a managed Kubernetes environment, you may not have access to the `etcd` cluster at all.

> **REMEMBER:** at the beginning, execute `export ETCDCTL_API=3`.

## Further Reading
- [Mastering etcd Backup in Kubernetes](https://medium.com/@Ibraheemcisse/mastering-etcd-backup-in-kubernetes-cka-ready-guide-55363642a60c)
- [Backing up an etcd cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
- [etcd recovery guide](https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md)
