# Helm Components

- **Official Documentation:** [Three Big Concepts](https://helm.sh/docs/intro/using_helm/#three-big-concepts)

Helm consists of several key components and concepts that work together to manage applications in Kubernetes.

```text
[Helm CLI] ---> Fetches [Chart] from [Repository / Artifact Hub]
      │
      └─── Combines Chart Templates + values.yaml
                 │
                 └─── Deploys to Cluster as a [Release]
                            │
                            ├── Creates/Updates [Revision 1, 2, ...]
                            └── Stores state as [Kubernetes Secrets]
```

---

## 1. Helm CLI (The Client)

- The command-line tool (`helm`) installed on your local machine or bastion host.
- Responsible for:
  - Interacting with repositories (`helm repo ...`).
  - Rendering templates with user configuration.
  - Making direct API calls to Kubernetes via your `~/.kube/config`.
  - Managing release lifecycles (`install`, `upgrade`, `rollback`, `uninstall`).

---

## 2. Helm Chart (The Package)

A **Chart** is a packaged collection of templated Kubernetes YAML manifests and metadata.

- **Templates:** Standard manifests (`deployment.yaml`, `service.yaml`) written with Go templating directives (e.g., `{{ .Values.image.repository }}`).
- **`values.yaml`:** The default input/settings file. Defines the configurable variables for the templates (replicas, image tags, port numbers).
- **Customization:** You rarely write charts from scratch in standard administration—you customize existing ones by overriding values in `values.yaml`.

---

## 3. Release & Revision (The Deployed Instance)

### What is a Release?
A **Release** is a specific running instance of a chart deployed into a Kubernetes cluster.

- **Why Release Names Matter:** 
  You can deploy the **same chart multiple times** into the same or different namespaces with different release names and configurations:
  ```bash
  # Production instance
  helm install prod-site bitnami/wordpress -n prod

  # Staging instance from the same chart
  helm install dev-site bitnami/wordpress -n dev
  ```
  Each release is tracked, updated, and deleted independently.

### What is a Revision?
Every time you install or upgrade a release (`helm upgrade`), Helm creates a **new Revision** (Revision 1, Revision 2, etc.):
- Revisions are snapshots of the release state.
- They allow instant rollbacks to a previous stable state:
  ```bash
  helm rollback <release-name> <revision-number>
  ```

---

## 4. Repositories & Artifact Hub

### Repositories (Repos)
- An HTTP server that holds an `index.yaml` and packaged chart archive files (`.tgz`).
- Examples of chart publishers: Bitnami, Prometheus Community, Ingress-NGINX.

### Artifact Hub (`artifacthub.io`)
- The central public registry/catalog where thousands of open-source Helm repositories are aggregated.
- Use it in your browser to find chart repository URLs and configuration documentation.

---

## 5. Release Metadata & Storage

How does Helm know what is installed in your cluster?

- Helm **does not** store release data on your local laptop.
- In **Helm 3**, all release metadata, chart contents, and revision history are stored **directly inside the Kubernetes cluster as Kubernetes Secrets** in the release's target namespace.
- **Why this matters:**
  - Any authorized team member using `helm` can see, update, and manage the same releases.
  - Metadata survives machine restarts and client changes as long as the cluster is alive.

```bash
# View Helm release secrets in a namespace
kubectl get secrets -n <namespace> -l owner=helm
```

---

## Component Quick Reference (CKA Summary)

| Component | Role | Where It Lives |
| :--- | :--- | :--- |
| **Helm CLI** | The tool you run commands with | Local machine / Bastion |
| **Chart** | The blueprint/template of the application | Repo or local directory |
| **values.yaml** | Default configurations for the chart | Inside the chart / overridden by user |
| **Release** | A running application instance | Kubernetes cluster |
| **Revision** | Versioned snapshot of a release | Stored as metadata |
| **Repository** | Web server storing packaged charts | Remote HTTP server / Artifact Hub |
| **Release Secrets**| Cluster state & history storage | Kubernetes Secrets in release namespace |
