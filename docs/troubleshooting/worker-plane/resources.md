# Worker Plane Failure: Resources

Check CPU, memory, disk, and process count on the node. Run these on the worker node itself.

```bash
# CPU and memory
top
free -h

# Disk space
df -h

# Number of processes (for PID pressure)
ps aux | wc -l
cat /proc/sys/kernel/pid_max
```

| Symptom | Condition | Fix |
| :--- | :--- | :--- |
| Memory is full | `MemoryPressure=True` | Free memory, stop unused workloads, or add memory |
| Disk is full | `DiskPressure=True` | Remove old images and logs, or add disk space |
| Too many processes | `PIDPressure=True` | Find and stop runaway processes |

To remove unused container images on a node that uses containerd:

```bash
crictl rmi --prune
```
