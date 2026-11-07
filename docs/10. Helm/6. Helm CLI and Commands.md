# Helm Basics (CLI & Release Management)

- **Official Documentation:** [Helm CLI Commands](https://helm.sh/docs/helm/helm/)

The Helm CLI (`helm`) is used to search for packages, manage chart repositories, and control the complete lifecycle of Kubernetes applications.

---

## 1. Getting Help in the CLI (Fastest Exam Lookup)

Never memorize flags blindly—Helm provides built-in help for every command:

```bash
# General help and list of top-level commands
helm --help

# Subcommand-specific help and available flags
helm repo --help
helm install --help
helm list --help
```

---

## 2. Searching for Charts

Helm allows you to search for charts in two locations:

| Command                      | Where It Searches                              | When to Use                                    |
| :--------------------------- | :--------------------------------------------- | :--------------------------------------------- |
| `helm search hub <keyword>`  | The public **Artifact Hub** (`artifacthub.io`) | When looking for new charts online             |
| `helm search repo <keyword>` | **Locally added repositories**                 | When finding charts in repos you already added |

```bash
# Search Artifact Hub for WordPress
helm search hub wordpress

# Search your local repository cache for Nginx
helm search repo nginx
```

---

## 3. Managing Repositories (`helm repo`)

Before installing a chart from a third-party source (e.g., Bitnami), you must add its repository:

```bash
# 1. Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# 2. List all added repositories
helm repo list

# 3. Update local chart cache (similar to 'apt update')
helm repo update

# 4. Remove a repository
helm repo remove bitnami
```

---

## 4. Installing Charts (`helm install`)

To deploy an application to your cluster as a new release:

```bash
# Basic syntax:
# helm install <RELEASE_NAME> <CHART_NAME> [FLAGS]

helm install my-web bitnami/nginx
```

#### Typical CKA Exam Scenario:

> **Question:** _"Deploy the Apache web server using the chart from Bitnami repository with the release name `amaze-surf`."_

```bash
# 1. Ensure the repo is added & updated
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 2. Deploy with the exact release name specified
helm install amaze-surf bitnami/apache

# (If a specific namespace was requested):
helm install amaze-surf bitnami/apache -n web --create-namespace
```

### Essential Exam Flags for `helm install`

| Flag                 | Purpose                                      | Example                                                     |
| :------------------- | :------------------------------------------- | :---------------------------------------------------------- |
| `-n, --namespace`    | Deploys release into a specific namespace    | `helm install web bitnami/nginx -n prod`                    |
| `--create-namespace` | Automatically creates namespace if missing   | `helm install web bitnami/nginx -n prod --create-namespace` |
| `--generate-name`    | Let Helm generate a unique release name      | `helm install bitnami/nginx --generate-name`                |
| `--dry-run`          | Simulates install without creating resources | `helm install test bitnami/nginx --dry-run`                 |
| `--set`              | Overrides default values on the fly          | `helm install web bitnami/nginx --set replicaCount=2`       |
| `-f, --values`       | Passes a custom YAML values file             | `helm install web bitnami/nginx -f custom-values.yaml`      |

---

## 5. Frequently Used Listing Commands

### 1. Listing Releases (`helm list` or `helm ls`)

> **CKA Warning:** By default, `helm list` only lists releases in the **current/default namespace** that are currently **deployed**. Always use `-A` if you are searching for releases across the cluster.

```bash
# List releases in current namespace
helm list

# List releases in a specific namespace
helm list -n prod

# List releases across ALL namespaces (high yield for CKA!)
helm list -A
# or
helm list --all-namespaces

# List releases with ALL statuses (deployed, failed, pending, uninstalled)
helm list -a
# or
helm list -A --all

# Filter by release status
helm list --failed
helm list --pending

# List only release names (quiet / scripting mode)
helm list -q

# Format output as JSON or YAML
helm list -o json
helm list -o yaml

# Sort releases by release date
helm list --date
```

### 2. Listing Release Revision History (`helm history`)

Shows all past revisions, deployment dates, chart versions, and status (superseded, deployed, failed):

```bash
# View revision history of a release
helm history my-web -n prod

# Short alias
helm hist my-web -n prod
```

### 3. Listing Repositories & Charts

```bash
# List all configured chart repositories
helm repo list

# List all available charts in configured repositories
helm search repo

# List all available versions of a specific chart (crucial when exam specifies a version!)
helm search repo bitnami/nginx -l
# or
helm search repo bitnami/nginx --versions
```

### 4. Inspecting Release Details (`helm status` & `helm get`)

```bash
# Show status, notes, and k8s resources belonging to a release
helm status my-web -n prod

# Show user-supplied custom values for a release
helm get values my-web -n prod

# Show ALL values (defaults + user overrides combined)
helm get values my-web -n prod --all

# Show the complete rendered Kubernetes YAML manifests applied to the cluster
helm get manifest my-web -n prod

# Show the release notes (instructions printed on install/upgrade)
helm get notes my-web -n prod
```

---

## 6. Uninstalling Releases (`helm uninstall`)

Deletes all Kubernetes objects (Deployments, Services, ConfigMaps, Secrets, etc.) created by the release:

```bash
# Syntax: helm uninstall <RELEASE_NAME> [-n <NAMESPACE>]
helm uninstall my-web -n prod
```

---

## 7. CKA Cheat Sheet: Helm Basics

```bash
# 1. Add repo & update
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 2. Search local repos
helm search repo nginx

# 3. Install in a specific namespace (creating it if needed)
helm install my-nginx bitnami/nginx -n web-ns --create-namespace

# 4. Check releases
helm list -A

# 5. Check release status
helm status my-nginx -n web-ns

# 6. Delete release and clean up all resources
helm uninstall my-nginx -n web-ns
```

---

## 8. Useful Helm CLI Commands Quick Reference Table

| Category | Command | Description |
| :--- | :--- | :--- |
| **Repo** | `helm repo add <name> <url>` | Add a remote chart repository |
| **Repo** | `helm repo list` | List all configured repositories |
| **Repo** | `helm repo update` | Refresh local cache of all repositories |
| **Repo** | `helm repo remove <name>` | Remove a configured repository |
| **Search** | `helm search repo <keyword>` | Search charts in locally added repositories |
| **Search** | `helm search repo <chart> -l` | List all available versions of a chart |
| **Search** | `helm search hub <keyword>` | Search charts publicly on Artifact Hub |
| **Install** | `helm install <name> <chart>` | Install a chart as a release |
| **Install** | `helm install <name> <chart> -n <ns> --create-namespace` | Install into a namespace, creating it if needed |
| **Install** | `helm install <name> <chart> --set key=val` | Install with on-the-fly parameter overrides |
| **Install** | `helm install <name> <chart> -f values.yaml` | Install with a custom values YAML file |
| **Install** | `helm install <name> <chart> --dry-run` | Simulate installation without creating resources |
| **List** | `helm list` | List releases in current namespace |
| **List** | `helm list -A` | List releases across **all namespaces** |
| **List** | `helm list -a` | List releases in all states (deployed, failed, pending) |
| **Inspect**| `helm status <name> [-n <ns>]` | Show release status, notes, and Kubernetes resources |
| **Inspect**| `helm show values <chart>` | Display the default `values.yaml` of a chart |
| **Inspect**| `helm get values <name> [-n <ns>]` | View custom values supplied to an existing release |
| **Inspect**| `helm get manifest <name> [-n <ns>]`| View rendered Kubernetes manifests applied to cluster |
| **History**| `helm history <name> [-n <ns>]` | View revision history of a release |
| **Lifecycle**| `helm upgrade <name> <chart>` | Upgrade an existing release |
| **Lifecycle**| `helm rollback <name> <revision>` | Roll back a release to a specific revision |
| **Delete** | `helm uninstall <name> [-n <ns>]` | Delete release and remove all its cluster resources |

