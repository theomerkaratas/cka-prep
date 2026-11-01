# Commands and Arguments

Understanding how Kubernetes overrides Docker commands and arguments is a frequent topic in the CKA exam.

---

## Docker vs. Kubernetes Mapping

| Docker Term | Kubernetes Equivalent | Purpose |
| :--- | :--- | :--- |
| **`ENTRYPOINT`** | **`command`** | The executable to run when the container starts. |
| **`CMD`** | **`args`** | The default parameters/arguments passed to the executable. |

---

## Dockerfile Example

```dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["10"]
```

Running this image in Docker:
```bash
# Runs 'sleep 10' (uses default CMD argument)
docker run my-sleeper

# Overrides CMD with '30', running 'sleep 30'
docker run my-sleeper 30

# Overrides ENTRYPOINT and CMD, running 'echo Hello'
docker run --entrypoint echo my-sleeper "Hello"
```

---

## Kubernetes Pod Specification

In a Kubernetes Pod definition, use `command` to override `ENTRYPOINT` and `args` to override `CMD`:

```yaml
# pod-sleeper.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
  - name: sleeper
    image: ubuntu
    command: ["sleep"] # Overrides Docker ENTRYPOINT
    args: ["3600"]      # Overrides Docker CMD
```

Alternative YAML list syntax:
```yaml
spec:
  containers:
  - name: sleeper
    image: ubuntu
    command:
    - "sleep"
    args:
    - "3600"
```

Running shell scripts or inline commands:
```yaml
spec:
  containers:
  - name: busybox-worker
    image: busybox
    command: ["/bin/sh", "-c"]
    args: ["echo Initializing... && sleep 60 && echo Done!"]
```

---

## CKA Rule on Immutability

> **Important:** Once a Pod is created, you **cannot modify its `command` or `args` in-place**. To change them, you must delete and recreate the Pod (or edit the parent Deployment):
> ```bash
> # Extract YAML, edit, and replace forcibly
> kubectl get pod my-pod -o yaml > pod.yaml
> # Edit pod.yaml, then:
> kubectl replace --force -f pod.yaml
> ```


