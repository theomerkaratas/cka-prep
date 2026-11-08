# Join the Other Masters (Multiple Masters)

Run these steps on `master2` and `master3`. Join one master at a time, and wait until each one shows `Ready` before you join the next.

## Join as a control plane node

Use the load balancer address, the token and hash from `kubeadm init`, and the `--control-plane` flag with the certificate key:

```bash
sudo kubeadm join <LB_IP_OR_DNS>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

Set up `kubectl` on each master in the same way as on `master1` (see [Initialize the first master](multi-master-init.md#set-up-kubectl)).

## If the token or certificate key has expired

- **Tokens** expire after 24 hours. Create a new join command on `master1`:

  ```bash
  sudo kubeadm token create --print-join-command
  ```

- **The certificate key** is deleted after 2 hours. Create a new one on `master1`:

  ```bash
  sudo kubeadm init phase upload-certs --upload-certs
  ```

  Use the printed key in the join command, with `--control-plane` as before.

## Next

Continue with [Join the worker nodes](join-workers.md), then [Verify the HA cluster](multi-master-verify.md).
