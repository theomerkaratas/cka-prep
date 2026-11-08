# Worker Plane Failure: kubelet Certificates

Check that the kubelet certificates are not expired, are in the right group, and are issued by the right CA.

## Check Expiry, Subject, and Issuer

The kubelet client certificate is usually at `/var/lib/kubelet/pki/kubelet-client-current.pem`:

```bash
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates -subject -issuer
```

- **Dates:** `notAfter` must be in the future.
- **Subject:** should include `O=system:nodes` and `CN=system:node:<node-name>`.
- **Issuer:** must match the cluster CA.

## Compare with the Cluster CA

```bash
openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -subject
```

The issuer of the kubelet certificate should match this CA subject.

## Check the kubelet Kubeconfig

```bash
cat /etc/kubernetes/kubelet.conf
```

Confirm the certificate and key paths in this file point to valid files.

## Renew if Expired

If the certificate has expired, check the rotation setting and restart the kubelet:

```bash
systemctl restart kubelet
```

Check the logs again with `journalctl -u kubelet` to confirm the certificate loaded.
