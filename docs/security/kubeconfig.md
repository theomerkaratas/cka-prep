# Understanding Kubeconfig & Kubernetes API Authentication

## 1. Core Concept: Communicating with the Kubernetes REST API
Everything in a Kubernetes cluster is exposed via a REST API. When you request a list of pods, you are essentially sending an HTTPS request to the `kube-apiserver`.

To authenticate your identity, the API server requires mutual TLS (mTLS) certificates.

If you were to make this request directly using `curl`, the command would look like this:

```sh
curl https://my-kube-playground:6443/api/v1/pods \
  --key admin.key \
  --cert admin.crt \
  --cacert ca.crt
```

- `--key`: Your private key.
- `--cert`: Your signed client certificate (your identity).
- `--cacert`: The cluster's Root CA certificate (used to verify that the API server is legitimate).

---

## 2. Doing the Same Thing via kubectl
`kubectl` is not magic; under the hood, it is simply a CLI wrapper that sends the same HTTPS requests to the API server.

Without a configuration file, you would have to pass all those connection flags manually to `kubectl` every single time:

```sh
kubectl get pods \
  --server https://my-kube-playground:6443 \
  --client-key admin.key \
  --client-certificate admin.crt \
  --certificate-authority ca.crt
```

---

## 3. The Problem: Repetitive and Error-Prone
Typing out all four flags (`--server`, `--client-key`, `--client-certificate`, `--certificate-authority`) for every single command (`kubectl get pods`, `kubectl describe node`, `kubectl logs`) is tedious, unmaintainable, and prone to typos.

---

## 4. The Solution: The kubeconfig File
To eliminate this repetitive work, Kubernetes uses configuration files known as `kubeconfig` files.

A kubeconfig file organizes these details into three main sections:
1. Clusters: Where the cluster is hosted (server URL and CA certificate).
2. Users: Who you are (client certificate and client key).
3. Contexts: Which user accesses which cluster (and optionally, which default namespace).

With a `kubeconfig` file, you can run:

`kubectl get pods --kubeconfig=/path/to/custom-config`

---

## 5. Default Location: ~/.kube/config
To avoid passing `--kubeconfig` with every command, `kubectl` automatically looks for a file named `config` located in the `.kube` directory inside the user's home directory:

`~/.kube/config` or `$HOME/.kube/config`

If your configuration file is placed there, `kubectl` reads it by default, allowing you to simply run:

```sh
kubectl get pods
```

---

## 6. Anatomy of a `Kubeconfig` File (Example)

Here is a full example of a standard `~/.kube/config` file:

```yaml
apiVersion: v1
kind: Config
preferences: {}

clusters:
  - name: Development
    cluster:
      server: https://dev.example.com:6443
      certificate-authority: /etc/kubernetes/pki/dev-ca.crt
  - name: Production
    cluster:
	  server: https://prod.example.com:6443
	  certificate-authority: /etc/kubernetes/pki/prod-ca.crt
  - name: Google
    cluster:
      server: https://gke.example.com:6443
      certificate-authority: /etc/kubernetes/pki/google-ca.crt

users:
  - name: Admin
    user:
      client-certificate: /etc/kubernetes/pki/admin.crt
      client-key: /etc/kubernetes/pki/admin.key
  - name: Dev User
	user:
	  client-certificate: /etc/kubernetes/pki/dev-user.crt
	  client-key: /etc/kubernetes/pki/dev-user.key
  - name: Prod User
	user:
	  client-certificate: /etc/kubernetes/pki/prod-user.crt
	  client-key: /etc/kubernetes/pki/prod-user.key

contexts:
  - name: Admin@Production
    context:
      cluster: Production
      user: Admin
      namespace: production # this wil lbe the default namespace
  - name: Dev@Google
    context:
      cluster: Google
      user: Dev User

# Currently active context
current-context: Admin@Production
```

or we can change namespaces with the commands below:

```sh
# Set the default namespace to 'production' for the 'Admin@Production' context:
kubectl config set-context Admin@Production --namespace=production

# Set the default namespace to 'development' for the 'Dev@Google' context:
kubectl config set-context Dev@Google --namespace=development

# Set the default namespace for the current context:
kubectl config set-context --current --namespace=production

# Set context in a specific file
k config use-context <context_name> --kubeconfig=/path/to/file
```

---

### Key Variations: File Paths vs. Embedded Base64 Data
In a `kubeconfig` file, certificates can be supplied in two ways:
1. File paths:
   - `certificate-authority: /path/to/ca.crt`
   - `client-certificate: /path/to/admin.crt`
   - `client-key: /path/to/admin.key`
1. Embedded Base64 data (recommended for portability):
   - `certificate-authority-data: <BASE64_STRING>`
   - `client-certificate-data: <BASE64_STRING>`
   - `client-key-data: <BASE64_STRING>`

---

## 7. Useful CKA `kubectl` config Commands

In the CKA exam, always prefer using `kubectl` config subcommands instead of manually modifying the YAML file:

```sh
# View the current configuration
kubectl config view
```

Example output (certificate data is redacted):

```yaml
apiVersion: v1
kind: Config
current-context: kubernetes-admin@kubernetes

clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://172.17.0.5:6443
  name: kubernetes

contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes

users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

```sh
# if the kubeconfig file in another location
kubectl config view --kubeconfig=/path/to/file

# View contexts
kubectl config get-contexts

# View the currently active context
kubectl config current-context

# Switch to a different context, it will also affect the file
kubectl config use-context Dev@Google

# Set a default namespace for the current context
kubectl config set-context --current --namespace=<namespace-name>

# View other options
kubectl config -h

# Change kubeconfig path
vi ~/.bashrc
export KUBECONFIG=/new/path
source ~/.bashrc
```

---

## 8. Kubeconfig Structural Relationship

```text
[ Context: Admin@Production ]
        │
        ├──> Cluster: Production  --> (Server: https://prod:6443, CA: prod-ca.crt)
        ├──> User: Admin          --> (Client Cert: admin.crt, Client Key: admin.key)
        └──> Namespace: production (Default namespace for commands)
```