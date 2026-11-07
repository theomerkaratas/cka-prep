# Helm Installation and Configuration

- **Official Documentation:** [Installing Helm](https://helm.sh/docs/intro/install/)

> **CKA Exam Tip:** In the actual CKA exam, Helm is almost always **pre-installed** on the nodes where it is needed. However, you should know how to verify it, quickly install it if missing, and configure auto-completion to save time.

---

## 1. Verify Existing Installation

Always check if Helm is already available in your terminal:

```bash
helm version
# or short version
helm version --short
```

---

## 2. Installation Methods

### Method 1: Automated Script (Recommended & Fastest)

Downloads the official installer script which detects the OS and architecture automatically:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

---

### Method 2: From Binary Release (Universal & Reliable)

Best when you need a specific version or when package managers are restricted:

```bash
# 1. Download the tarball (adjust version and architecture if needed)
wget https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz

# 2. Unpack
tar -zxvf helm-v3.16.2-linux-amd64.tar.gz

# 3. Move binary to system PATH
sudo mv linux-amd64/helm /usr/local/bin/helm

# 4. Clean up
rm -rf helm-v3.16.2-linux-amd64.tar.gz linux-amd64
```

---

### Method 3: Package Managers

#### Debian / Ubuntu (`apt`):
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm -y
```

#### Snap:
```bash
sudo snap install helm --classic
```

#### macOS (Homebrew - for local study):
```bash
brew install helm
```

---

## 3. Configuration & Auto-completion (Time Saver for CKA)

Just like `kubectl`, enabling auto-completion for `helm` saves valuable exam minutes:

### Temporary (current shell session):
```bash
source <(helm completion bash)
```

### Persistent (added to profile):
```bash
echo "source <(helm completion bash)" >> ~/.bashrc
```

---

## 4. Connecting Helm to Kubernetes

Helm v3 requires **no server-side configuration**. It automatically uses:
- The default kubeconfig at `~/.kube/config`
- Or the path specified by the `KUBECONFIG` environment variable:
  ```bash
  export KUBECONFIG=/path/to/custom/kubeconfig
  ```
- Or passed via the `--kubeconfig` flag:
  ```bash
  helm list --kubeconfig /etc/kubernetes/admin.conf
  ```

