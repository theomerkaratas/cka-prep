# Worker Plane Failure: kubelet

The kubelet runs on every node and keeps pods running. If it is down, the node becomes `NotReady`.

## Check the Service

```bash
service kubelet status
systemctl status kubelet
```

A healthy kubelet shows `Active: active (running)`. To start it if it is stopped:

```bash
systemctl start kubelet
systemctl enable kubelet
```

## Check the Logs

```bash
journalctl -u kubelet --no-pager | tail -50
journalctl -u kubelet -f
```

## Check the Configuration

The kubelet config file is usually at `/var/lib/kubelet/config.yaml`. The kubelet service file is at `/etc/systemd/system/kubelet.service.d/`. After a change, reload and restart:

```bash
systemctl daemon-reload
systemctl restart kubelet
```

Next: [kubelet Certificates](kubelet-certificates.md).
