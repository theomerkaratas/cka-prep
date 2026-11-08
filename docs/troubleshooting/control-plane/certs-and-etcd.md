# Control Plane Failure: Certificates and etcd

## Certificate expiry

```bash
kubeadm certs check-expiration
```

## etcd health

```bash
ETCDCTL_API=3 etcdctl endpoint health \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key
```
