# Docker Storage Drivers, Layer Architecture & File Systems

Docker uses **Storage Drivers** to implement a layered, union file system with **Copy-on-Write (CoW)** for image layers and container writable layers, while persisting data beyond container lifecycles using **Volume Mounts** and **Bind Mounts**.

---

## 1. Docker Host Storage Layout (`/var/lib/docker`)

When the Docker daemon runs on a Linux host, it organizes runtime state in subdirectories under `/var/lib/docker`:

| Directory | Purpose |
| :--- | :--- |
| `/var/lib/docker/image/` | Stores metadata and content hashes for read-only image layers |
| `/var/lib/docker/containers/` | Stores metadata, configs, and stdout/stderr logs for active containers |
| `/var/lib/docker/volumes/` | Default location for Docker-managed persistent volumes |
| `/var/lib/docker/<driver>/` | Stores underlying layer data according to the active driver (`overlay2`, `aufs`, etc.) |

---

## 2. Docker Image & Container Layered Architecture

Docker images and containers are composed of stacked union filesystem layers:

```
+-----------------------------------------------------------+
| Container Writable Layer (R/W)  [Logs, temp files, delta] | <-- Ephemeral 
+-----------------------------------------------------------+      (dies 
| Image Layer 3 (RO): Application code (app.py)             |       with 
+-----------------------------------------------------------+     container)
| Image Layer 2 (RO): Runtime dependencies (Python, pip)    | <-- Immutable 
+-----------------------------------------------------------+     & Cached
| Image Layer 1 (RO): Base OS Layer (Ubuntu / Alpine)       |
+-----------------------------------------------------------+
```

### A. Read-Only (RO) Image Layers
- Each instruction in a `Dockerfile` (`RUN`, `COPY`, `ADD`) creates an immutable, read-only layer containing only the diff from the previous layer.
- Layers are cached and reused across multiple images and containers to save disk space and accelerate builds.

### B. Read-Write (R/W) Container Layer
- Created on top of the image stack when `docker run` is executed.
- Stores all runtime mutations (log outputs, modified files, temporary data).
- **Ephemeral Lifecycle:** When the container is removed (`docker rm`), this R/W layer and all its uncommitted data are permanently destroyed.

### C. Copy-on-Write (CoW) Mechanism
- Files in the lower image layers are strictly read-only.
- If a running container modifies an existing file baked into the image (e.g., `app.py`), the storage driver **copies the file from the read-only image layer up into the writable container layer**.
- Subsequent reads/writes target this writable copy, leaving the underlying base image layer unchanged for other containers sharing that image.

---

## 3. Persisting Data: Volume Mounts vs. Bind Mounts

To ensure data survives container deletion, storage must be mounted outside the ephemeral R/W layer.

```
       Volume Mount                                    Bind Mount
+-------------------------+                     +-------------------------+
|    Docker Container     |                     |    Docker Container     |
|   (/var/lib/mysql)      |                     |   (/var/lib/mysql)      |
+------------+------------+                     +------------+------------+
             |                                               |
             v                                               v
+-------------------------+                     +-------------------------+
|   Managed by Docker     |                     |  Any Host Directory     |
| /var/lib/docker/volumes |                     |       /data/mysql       |
+-------------------------+                     +-------------------------+
```

### Comparison Matrix

- Who owns the folder which hold the data?

| Feature | Volume Mount (`volume`) | Bind Mount (`bind`) |
| :--- | :--- | :--- |
| **Source on Host** | Managed by Docker (`/var/lib/docker/volumes/<vol-name>/_data`) | Arbitrary host path (e.g., `/data/mysql`) |
| **Creation** | Explicit via CLI or automatically initialized by Docker | Must exist on host or is created as host directory |
| **Docker CLI Management** | Managed via `docker volume ...` commands | Bypasses Docker volume management tools |
| **Portability** | High (independent of host directory layout) | Low (relies on specific host filesystem paths) |
| **K8s Equivalent Concept** | Dynamic PV / PVC backed storage | `hostPath` volume |

---

## 4. Mounting Syntax: Legacy (`-v`) vs. Modern (`--mount`)

### A. Volume Mounts

```bash
# 1. Pre-create the volume (recommended)
docker volume create data_volume

# 2. Run with legacy -v flag (auto-creates volume if missing)
docker run -d --name db -v data_volume:/var/lib/mysql mysql

# 3. Run with preferred --mount flag (explicit key=value format)
docker run -d --name db \
  --mount type=volume,source=data_volume,target=/var/lib/mysql \
  mysql
```

### B. Bind Mounts

```bash
# Run with legacy -v flag (requires absolute host path)
docker run -d --name db -v /data/mysql:/var/lib/mysql mysql

# Run with preferred --mount flag
docker run -d --name db \
  --mount type=bind,source=/data/mysql,target=/var/lib/mysql \
  mysql
```

> ⚠️ **Important Caveats (Exam & Production Gotchas):**
> - **Relative vs. Absolute:** `-v data_volume:/path` is a **volume mount**; `-v /data/mysql:/path` (starting with `/`) is a **bind mount**. Confusing the two causes subtle host permission or pathing failures.
> - **`--mount` Strictness:** If using `--mount type=bind`, Docker will produce an error if the host `source` directory does not exist. The older `-v` syntax automatically creates a missing host directory as root.

---

## 5. Storage Drivers Overview

Storage drivers manage how layers are stacked, how CoW operates, and how files are presented to containers.

| Driver | Description / Ecosystem Notes |
| :--- | :--- |
| **`overlay2`** | Current industry standard and default for modern Linux distros (Ubuntu, Debian, RHEL, CentOS). Fast, efficient page-cache sharing. |
| **`AUFS`** | Older union filesystem, formerly default on early Ubuntu releases. |
| **`device mapper`** | Block-level driver historically used on CentOS/RHEL before kernel overlay support matured. |
| **`Btrfs` / `ZFS`** | Filesystem-level drivers that handle CoW natively at the filesystem layer. |

---

## 6. Inspection & Operational Verification Commands

```bash
# Check the active storage driver and root directory on the host
docker info | grep -E "Storage Driver|Docker Root Dir"

# Volume management commands
docker volume create <vol-name>
docker volume ls
docker volume inspect <vol-name>
docker volume rm <vol-name>
docker volume prune   # Removes all unused local volumes

# Inspect container mount details
docker inspect <container-name-or-id> --format '{{json .Mounts}}' | jq .

# Verify volume files directly on Linux host filesystem
ls -la /var/lib/docker/volumes/<vol-name>/_data
```