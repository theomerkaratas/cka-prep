# Kubernetes Image Security & Private Registry Configuration

When deploying Pods in Kubernetes, container images often reference publicly accessible registries (e.g., `image: nginx` on Docker Hub). However, in enterprise and production environments, internal applications and proprietary code are hosted in authenticated **Private Container Registries** (such as AWS ECR, Google Artifact Registry, Azure Container Registry, Harbor, Nexus, or private Docker Hub repositories).

---

## 1. Anatomy of a Container Image Name

When an image is declared simply as `nginx`, the container runtime resolves it into a fully qualified domain path following a structured hierarchy:

```text
[REGISTRY_DOMAIN] / [ACCOUNT_OR_ORG] / [IMAGE_NAME] : [TAG]
```

| Specified Image                | Fully Qualified Resolution       | Explanation                                                                                     |
| ------------------------------ | -------------------------------- | ----------------------------------------------------------------------------------------------- |
| `nginx`                        | `docker.io/library/nginx:latest` | Defaults to `docker.io` (Docker Hub), `library` (official maintained images), and `latest` tag. |
| `myuser/my-app:1.0`            | `docker.io/myuser/my-app:1.0`    | Explicit user namespace under Docker Hub.                                                       |
| `quay.io/coreos/etcd:v3.5`     | `quay.io/coreos/etcd:v3.5`       | Explicit third-party public registry (`quay.io`).                                               |
| `myreg.company.com/fin/api:v2` | `myreg.company.com/fin/api:v2`   | Private enterprise registry requiring authentication.                                           |

If an image resides in a private registry and no credentials are provided to the cluster, the Pod will fail to start, reporting **`ErrImagePull`** or entering an **`ImagePullBackOff`** loop.

---

## 2. Authentication Mechanism: Docker CLI vs. Kubernetes

On a local developer machine, authentication is interactive:
```bash
docker login myregistry.company.com
# Credentials are base64-encoded and stored locally in ~/.docker/config.json
```

In a Kubernetes cluster, worker node runtimes (such as `containerd` or CRI-O) run non-interactively. The node's `kubelet` cannot launch an interactive login session. Instead, Kubernetes stores the registry credentials securely as an internal **Secret** and passes them to the runtime during image pull operations.

This is handled by a specialized Secret type: **`kubernetes.io/dockerconfigjson`** (represented in CLI commands as `docker-registry`).

---

## 3. Step-by-Step Configuration

### Step 1: Create the Registry Secret

Use the `kubectl create secret docker-registry` command to generate the credential Secret:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \
  --docker-username=registry-user \
  --docker-password=registry-password \
  --docker-email=registry-user@org.com \
  --namespace=default
```

- **`regcred`**: The name assigned to the Secret object.
- **`--docker-server`**: The FQDN of the private registry (e.g., `private-registry.io`, `https://index.docker.io/v1/`, or `<account-id>.dkr.ecr.<region>.amazonaws.com`).
- **`--namespace`**: The target namespace. **Key requirement:** Secrets are namespaced; the Secret must reside in the exact same namespace as the Pod referencing it.

#### Alternative: Creating from an existing `~/.docker/config.json`
If you have already authenticated via `docker login` locally, you can create the Secret directly from your existing config file:

```bash
kubectl create secret generic regcred \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson \
  --namespace=default
```

---

### Step 2: Reference the Secret in the Pod Spec (`imagePullSecrets`)

Attach the Secret to the Pod manifest using the **`spec.imagePullSecrets`** array:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: default
spec:
  containers:
    - name: nginx
      image: private-registry.io/apps/internal-app:1.0
  imagePullSecrets:
    - name: regcred # Matches the Secret created above
```

### Execution Lifecycle:
1. The Pod is scheduled and assigned to a worker node.
2. The `kubelet` detects the `imagePullSecrets` configuration before issuing the pull request.
3. `kubelet` retrieves and decodes the credentials from `regcred`.
4. An authenticated request is sent to `private-registry.io`, the container image layers are downloaded, and the container starts.

---

## 4. Cluster Pattern: Automatic Injection via ServiceAccount

Adding `imagePullSecrets` to every individual Pod or Deployment manifest introduces operational overhead and increases configuration drift risk. 

To automate credential assignment, bind the Secret directly to the **ServiceAccount** governing the workload:

### Declarative Manifest:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
imagePullSecrets:
  - name: regcred
```

### Imperative Patch:
```bash
kubectl edit serviceaccount default -n default
# or shorthand:
kubectl edit sa default -n default

# Inside the editor (e.g., `vim` or `nano`), add the `imagePullSecrets` block at the root level (aligned with `metadata`, `apiVersion`, and `kind`):
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
  # ... other managed fields (creationTimestamp, uid, etc.)
imagePullSecrets:    # <-- ADD THESE TWO LINES
  - name: regcred    # <--
```

**Result:** Any Pod using this ServiceAccount automatically inherits the `imagePullSecrets` configuration without needing explicit declarations inside its own Pod specification.

---

## 5. Troubleshooting Common Issues

| Status / Symptom                                | Root Cause                                    | Resolution                                                                                                                               |
| ----------------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `ImagePullBackOff` / `401 Unauthorized`         | Invalid username, password, or server address | Inspect failure details via `kubectl describe pod <pod-name>` under the `Events` section. Re-create the Secret with correct credentials. |
| `Warning Failed ... secret "regcred" not found` | Namespace mismatch                            | Verify that both the Pod and Secret are deployed within the same namespace (`kubectl get secrets -n <namespace>`).                       |
| `ErrImagePull` / `manifest unknown`             | Typo in repository name or tag                | Confirm image existence and exact tag syntax on the remote registry (`<registry>/<repo>/<image>:<tag>`).                                 |