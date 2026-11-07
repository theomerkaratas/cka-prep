# etcd

`etcd` is a strongly consistent, distributed key-value data store used by Kubernetes to store all cluster data, state, and specifications.

- When we run etcd via `./etcd`, it starts a service listening on port **2379** by default (peer communication uses port **2380**).
- The default CLI client bundled with etcd is `etcdctl`.

---

## Basic `etcdctl` Commands

```bash
# Store a key-value pair
./etcdctl put key1 value1

# Retrieve a value by key
./etcdctl get key1

# View help and all available commands
./etcdctl --help
```

Common commands in `etcdctl`:
```text
COMMANDS:
        get                 Gets the key or a range of keys
        put                 Puts the given key into the store
        del                 Removes the specified key or range of keys [key, range_end]
        txn                 Txn processes all the requests in one transaction
        compaction          Compacts the event history in etcd
        alarm disarm        Disarms all alarms
        alarm list          Lists all alarms
        defrag              Defragments the storage of the etcd members with given endpoints
        endpoint health     Checks the healthiness of endpoints specified in --endpoints flag
```

---

## Listing Keys Stored by Kubernetes

Kubernetes stores all its objects under the `/registry` prefix:

```bash
kubectl exec etcd-controlplane -n kube-system -- etcdctl \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only
```

Sample output:
```text
/registry/apiregistration.k8s.io/apiservices/v1.
/registry/apiregistration.k8s.io/apiservices/v1.apps
/registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.autoscaling
/registry/apiregistration.k8s.io/apiservices/v1.batch
/registry/apiregistration.k8s.io/apiservices/v1.networking.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.rbac.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.storage.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1beta1.admissionregistration.k8s.io
```

---

## etcdctl API Versions (v2 vs v3)

`etcdctl` supports two API versions: **v2** and **v3**.

### v2 Commands
```bash
etcdctl backup
etcdctl cluster-health
etcdctl mk
etcdctl mkdir
etcdctl set
```

### v3 Commands
```bash
etcdctl snapshot save <file-path>
etcdctl endpoint health
etcdctl get <key>
etcdctl put <key> <value>
```

To use API version 3, set the `ETCDCTL_API` environment variable:
```bash
export ETCDCTL_API=3
```
> When `ETCDCTL_API` is not set, older versions may default to version 2. In modern Kubernetes, API version 3 is always used.

---

## Authenticating with Certificates

In a secure cluster, `etcdctl` must authenticate using TLS certificates:

```bash
kubectl exec etcd-controlplane -n kube-system -- sh -c \
  "ETCDCTL_API=3 etcdctl get / \
  --prefix --keys-only --limit=10 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key"
```

