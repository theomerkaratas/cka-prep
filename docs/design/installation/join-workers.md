# Join the Worker Nodes

Worker nodes join the cluster with a token and a CA certificate hash. Get the join command from a control plane node:

```bash
kubeadm token create --print-join-command
```

Run the printed command with `sudo` on each worker node. It has this form:

```bash
sudo kubeadm join <CONTROL_PLANE_ENDPOINT>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

- **Single master:** `<CONTROL_PLANE_ENDPOINT>` is the master node's IP.
- **Multiple masters:** `<CONTROL_PLANE_ENDPOINT>` is the load balancer address. Do not use a single master's IP.

Tokens expire after 24 hours. If the join command fails with an expired token, run `kubeadm token create --print-join-command` again.

## Taint on a single-node cluster

If the cluster has only the master node and no workers, the master is tainted and will not run your Pods. Remove the taint so the master can schedule Pods:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## Next

- Single master: [Verify the cluster](single-master-verify.md)
- Multiple masters: [Verify the HA cluster](multi-master-verify.md)
