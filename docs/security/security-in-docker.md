# Fundamentals of Docker & Linux Security (Prerequisites for Kubernetes SecurityContext)

To fully understand Kubernetes `securityContext`, you first need to understand how containers run, share, and restrict resources on top of the underlying Linux kernel.

This foundational model rests on three core pillars:
1. **Isolation Mechanism: Linux Namespaces & Process ID (PID)**
2. **User Management: Container Root vs. Host Root**
3. **Privilege Granularity: Linux Capabilities & Privileged Mode**

---

## 1. Virtual Machines vs. Containers: How Isolation Works

- **Virtual Machines (VMs):** Each VM runs its own full guest operating system, independent kernel, and virtualized hardware layer.
- **Containers:** Containers do **not** have their own kernel. They share the host machine's (the Node's) Linux kernel directly.

Because all containers share the same kernel, isolation is achieved via **Linux Namespaces** (PID, Mount, Network, IPC, UTS, and User namespaces).

### Process Isolation in Action
Consider running a container on a host: `docker run ubuntu sleep 3600`.

1. **Inside the Container (`ps aux`):**
   - The container operates within its own isolated PID namespace.
   - It cannot see processes running on the host or in other containers.
   - The `sleep 3600` process appears as **PID 1**.
2. **On the Host Machine (`ps aux`):**
   - The host kernel sees all processes across all child namespaces.
   - The exact same `sleep 3600` process is visible as a standard Linux process, but with an assigned host PID (e.g., **PID 4582**).

> **Key Takeaway:** A container is simply a standard Linux process running on the host kernel, restricted from seeing outside its assigned namespaces.

---

## 2. User Management: Container Root vs. Host Root

By default, unless configured otherwise in the image, Docker executes container processes as the **`root` user (UID 0)**.

### Why Running as Root Presents a Risk
If a vulnerability or container breakout occurs while a process runs as UID 0, an attacker can gain host-level root control. For defense-in-depth, security best practices dictate running container workloads as **non-root** users.

### Two Ways to Enforce a Non-Root User:

1. **Build Time (Dockerfile):**
   ```dockerfile
   FROM ubuntu
   USER 1000
   CMD ["sleep", "3600"]
   ```
2. **Run Time (CLI Flag):**
   ```bash
   docker run --user=1000 ubuntu sleep 3600
   ```

---

## 3. Linux Capabilities: What Can the Container Root Actually Do?

In traditional Linux architectures, system access was binary:
- **Unprivileged Users:** Subject to standard filesystem and system call restrictions.
- **Root User (UID 0):** Unrestricted power (could wipe storage, reconfigure networking, reboot the machine, modify system time, etc.).

Does the `root` user inside a container have unrestricted power over the host?  
**No.** The kernel breaks down traditional root authority into modular units called **Linux Capabilities**.

### What Are Linux Capabilities?
Capabilities divide monolithic superuser privileges into specific, fine-grained access rights:
- `CAP_CHOWN`: Change file ownership.
- `CAP_NET_BIND_SERVICE`: Bind a socket to privileged system ports (< 1024).
- `CAP_SYS_TIME`: Modify the system clock.
- `CAP_SYS_BOOT`: Reboot the host system.
- `CAP_NET_ADMIN`: Reconfigure network interfaces and routing tables.

### Docker's Default Behavior
Even when running as `root`, Docker starts containers with a **restricted subset of capabilities** by default:
- It **drops** dangerous capabilities like `CAP_SYS_BOOT`, `CAP_NET_ADMIN`, and `CAP_SYS_TIME`.
- A container root process cannot reboot the underlying host or disrupt neighboring containers.

### Customizing Capabilities in Docker:
You can fine-tune privileges during runtime using `--cap-add` and `--cap-drop`:

```bash
# Add network administration capability (e.g., for VPN or routing tools):
docker run --cap-add=NET_ADMIN ubuntu

# Strip away specific default privileges (e.g., prevent changing file ownership):
docker run --cap-drop=CHOWN ubuntu

# Run in full privileged mode (disables capability stripping and device restrictions):
docker run --privileged ubuntu
```

> ⚠️ The `--privileged` flag bypasses capability filtering and grants the container virtually unrestricted access to the host kernel and hardware devices. It should be avoided in production unless strictly necessary.

---

## Connecting to Kubernetes: The Link to `securityContext`

The Kubernetes **`securityContext`** field is a direct YAML abstraction of these exact Linux and container runtime security primitives:

| Docker / Linux Concept | Kubernetes Manifest Field |
|---|---|
| `--user=1000` | `securityContext.runAsUser: 1000` |
| Block execution as root | `securityContext.runAsNonRoot: true` |
| `--cap-add=NET_ADMIN` | `securityContext.capabilities.add: ["NET_ADMIN"]` |
| `--cap-drop=ALL` | `securityContext.capabilities.drop: ["ALL"]` |
| `--privileged` | `securityContext.privileged: true` |