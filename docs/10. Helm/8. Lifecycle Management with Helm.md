# Lifecycle Management with Helm

- **Official Documentation:** [Upgrading a Release and Recovering on Failure](https://helm.sh/docs/intro/using_helm/#helm-upgrade-and-helm-rollback-upgrading-a-release-and-recovering-on-failure)

Lifecycle management in Helm refers to tracking, updating, rolling back, and retiring applications across their lifespan in Kubernetes.

Every modification to a release increments its **Revision** number, creating an auditable history of the application.

```text
Install (Rev 1) ───► Upgrade (Rev 2) ───► Rollback to 1 (Rev 3)
```

---

## 1. Installing a Specific Chart Version

In the CKA exam, you are often asked to deploy a specific version of a chart rather than the latest:

```bash
# Search available versions first
helm search repo bitnami/nginx -l

# Install a specific version using --version
helm install my-web bitnami/nginx --version 15.2.0 -n prod
```

---

## 2. Upgrading a Release (`helm upgrade`)

When an application needs an update (new container image, new chart version, or modified configurations):

```bash
# Basic upgrade syntax:
helm upgrade <RELEASE_NAME> <CHART_NAME> [FLAGS]
```

### Basic Upgrade Scenarios (Standard Upgrades)

```bash
# Scenario 1: Upgrade to the latest chart version (standard upgrade)
helm upgrade my-web bitnami/nginx -n prod

# Scenario 2: Upgrade a parameter on the fly (e.g. scale replicas)
helm upgrade my-web bitnami/nginx --set replicaCount=3 -n prod

# Scenario 3: Upgrade using a new values file
helm upgrade my-web bitnami/nginx -f updated-values.yaml -n prod
```

### Upgrading to a Specific Version

#### Case A: Upgrade to a Specific Chart Version (`--version`)

```bash
# 1. Search available chart versions
helm search repo bitnami/nginx -l

# 2. Upgrade release to that specific chart version
helm upgrade my-web bitnami/nginx --version 15.4.0 -n prod
```

#### Case B: Upgrade to a Specific App / Container Image Version

If you want to update just the underlying application container image without changing chart versions:

```bash
# Use --reuse-values to preserve existing settings and update the image tag
helm upgrade my-web bitnami/nginx \
  --reuse-values \
  --set image.tag="1.25.3" \
  -n prod
```

#### Case C: Typical CKA Exam Scenario (Specific Chart Version + App Version)

> **Scenario:** *"Upgrade release `my-web` in namespace `prod` to use chart version `18.3.6` from the `bitnami` repository and Nginx image tag `1.27.2`. Ensure the new version is actively running."*

```bash
# 1. Upgrade chart version and app image tag together
helm upgrade my-web bitnami/nginx \
  --version 18.3.6 \
  --set image.tag="1.27.2" \
  --reuse-values \
  -n prod

# 2. Wait for the deployment rollout to complete
kubectl rollout status deployment/my-web -n prod

# 3. Verify the running Nginx version inside the pod
kubectl exec -n prod deployment/my-web -- nginx -v
```

### Verifying the Upgrade

```bash
# 1. Check Helm bumped to a new revision with the new version
helm history my-web -n prod

# 2. Check the active container image deployed in Kubernetes
kubectl get deployment my-web -n prod -o jsonpath='{.spec.template.spec.containers[*].image}'
```

### Powerful Upgrade Flags (High-Yield for CKA)

| Flag                 | What It Does                                                                                           | Example                                                                 |
| :------------------- | :----------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------- |
| **`--install`**      | If release exists, **upgrades** it; if it doesn't exist, **installs** it. Ideal for automated scripts. | `helm upgrade --install my-web bitnami/nginx -n prod`                   |
| **`--reuse-values`** | Keeps existing custom values and applies only new `--set` changes.                                     | `helm upgrade my-web bitnami/nginx --reuse-values --set replicaCount=4` |
| **`--reset-values`** | Discards previous user values and resets back to the chart's defaults.                                 | `helm upgrade my-web bitnami/nginx --reset-values`                      |
| **`--atomic`**       | If the upgrade fails, Helm automatically **rolls back** to the previous revision and waits for pods.   | `helm upgrade my-web bitnami/nginx --atomic`                            |

---

## 3. Viewing Release History (`helm history`)

Each install, upgrade, or rollback creates a new versioned snapshot called a **Revision**:

```bash
helm history my-web -n prod
```

### Example Output:

```text
REVISION  UPDATED                   STATUS          CHART         APP VERSION  DESCRIPTION
1         Sat Nov  7 10:00:00 2026  superseded      nginx-15.2.0  1.25.1       Install complete
2         Sat Nov  7 10:30:00 2026  deployed        nginx-15.4.0  1.25.3       Upgrade complete
```

- **`superseded`**: A previous revision that has been replaced.
- **`deployed`**: The currently active revision running in the cluster.
- **`failed`**: A release revision where deployment failed.

---

## 4. Rolling Back a Release (`helm rollback`)

If an upgrade introduces a bug or crashes your application, you can instantly revert to any previous revision.

```bash
# Syntax: helm rollback <RELEASE_NAME> [REVISION_NUMBER] [-n <NAMESPACE>]

# Roll back to Revision 1
helm rollback my-web 1 -n prod

# Roll back to the immediately preceding revision (omitting revision number)
helm rollback my-web -n prod
```

### Critical CKA Exam Concept: How Rollback Actually Works

> **Exam Note:** Rolling back to revision 1 **does not** delete revision 2 or revert the clock to revision 1.
> Instead, Helm creates a **NEW revision (Revision 3)** whose configuration is an exact duplicate of Revision 1!

```text
Revision 1: nginx-15.2.0 (superseded)
Revision 2: nginx-15.4.0 (superseded)
Revision 3: nginx-15.2.0 (deployed - Rollback to 1)
```

---

## 5. Important Limitation: Data vs Manifests

- **What Helm tracks and rolls back:** Kubernetes manifests and configuration (Deployments, Services, ConfigMaps, Secrets, replica counts).
- **What Helm DOES NOT roll back:** Application data stored on **PersistentVolumes (PVs)** or external databases. A rollback will restore the database pod's container version, but will not undo SQL statements or restored database records.

---

## 6. Lifecycle Management Cheat Sheet (CKA Quick Reference)

```bash
# 1. Install specific version
helm install my-app bitnami/nginx --version 15.2.0 -n web --create-namespace

# 2. Upgrade release with new parameter
helm upgrade my-app bitnami/nginx --set replicaCount=4 -n web

# 3. Check revision history
helm history my-app -n web

# 4. Roll back to revision 1 if issues occur
helm rollback my-app 1 -n web

# 5. Verify current status
helm status my-app -n web

# 6. Complete cleanup
helm uninstall my-app -n web
```
