# Helm Introduction

- **Official Documentation:** [Helm Documentation](https://helm.sh/docs/)

Helm is the **package manager for Kubernetes** (similar to `apt` or `yum` for Linux).

---

## Why Helm?

In standard Kubernetes, deploying a complete application often requires multiple YAML manifests:
- `deployment.yaml`
- `service.yaml`
- `configmap.yaml`
- `ingress.yaml`

Managing, versioning, and deploying these individually across different environments (dev, staging, prod) is error-prone. 

**Helm solves this by:**
- Packaging multiple Kubernetes manifests into a single reusable unit.
- Parameterizing configurations using templates (`values.yaml`).
- Tracking versions and enabling easy upgrades or rollbacks.

---

## Core Concepts (CKA Quick Reference)

| Concept | What It Is | Analogy |
| :--- | :--- | :--- |
| **Chart** | A bundle of templated Kubernetes YAML files and default values. | An `apt`/`rpm` package |
| **Repository** | An HTTP server where charts are stored and shared. | An `apt` repository |
| **Release** | A running instance of a chart deployed into a Kubernetes cluster. | An installed application |

---

## Helm Architecture (Helm 3)

For the CKA exam, keep these key architectural points in mind:

- **Client-only architecture:** Helm v3 interacts directly with the Kubernetes API server using your local `kubeconfig` (`~/.kube/config`).
- **No Tiller:** Helm v2 required a cluster-side component called `Tiller` (security risk). In **Helm v3, Tiller is completely removed**.
- **RBAC:** Helm inherits the exact user permissions defined in your `kubeconfig`.
- **Release Tracking:** Helm stores release information as Kubernetes **Secrets** inside the release's target namespace.

