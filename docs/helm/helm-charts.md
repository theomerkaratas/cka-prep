# Helm Charts

- **Official Documentation:** [Helm Charts Guide](https://helm.sh/docs/topics/charts/)

A **Helm Chart** is a structured collection of files that describe a related set of Kubernetes resources. Think of it as the source code or package definition for your application in Kubernetes.

---

## 1. Chart Directory Structure

When you create a chart with `helm create my-chart` or unpack an existing one, it follows this standard layout:

```text
my-chart/
├── Chart.yaml          # Mandatory: Metadata about the chart (name, version, etc.)
├── values.yaml         # Mandatory: Default configuration values for the templates
├── templates/          # Mandatory: Directory containing Kubernetes manifest templates
│   ├── deployment.yaml # Manifest template with Go template variables
│   ├── service.yaml    # Service template
│   ├── _helpers.tpl    # Partial templates and helper snippets
│   └── NOTES.txt       # User guide text displayed after running 'helm install'
├── charts/             # Optional: Directory for chart dependencies (subcharts)
├── README.md           # Optional: Human-readable documentation
└── LICENSE             # Optional: License information
```

---

## 2. Anatomy of `Chart.yaml`

The `Chart.yaml` file defines the chart's identity, metadata, and dependencies.

### Example `Chart.yaml`

```yaml
apiVersion: v2
name: my-web-app
version: 1.2.0
appVersion: "2.5.1"
description: A production-ready web application chart for Kubernetes
type: application
keywords:
  - web
  - nginx
  - frontend
home: https://example.com/my-web-app
sources:
  - https://github.com/example/my-web-app
maintainers:
  - name: Jane Doe
    email: jane@example.com
    url: https://janedoe.dev
icon: https://example.com/assets/icon.png
dependencies:
  - name: redis
    version: 17.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### Field-by-Field Breakdown

| Field | Required? | Purpose / Description |
| :--- | :---: | :--- |
| **`apiVersion`** | **Yes** | The schema version of the chart. Must be **`v2` for Helm 3** (`v1` is legacy Helm 2). |
| **`name`** | **Yes** | The name of the chart (e.g., `my-web-app`, `wordpress`). |
| **`version`** | **Yes** | The version of the **Chart package itself** following [SemVer 2](https://semver.org/) (e.g., `1.2.0`). |
| **`appVersion`** | No | The version of the **underlying application / container image** being deployed (e.g., `"2.5.1"`). Quotes recommended. |
| **`description`** | No | A single-sentence summary of what this chart installs. |
| **`type`** | No | Chart classification: **`application`** (deployable app, default) or **`library`** (reusable template helpers). |
| **`keywords`** | No | A list of search terms used when indexing the chart on registries like Artifact Hub. |
| **`home`** | No | URL to the official project or chart homepage. |
| **`sources`** | No | List of Git repo URLs or source code locations. |
| **`maintainers`** | No | List of contact objects (`name`, `email`, `url`) for the chart owners. |
| **`icon`** | No | An SVG or PNG image URL used as the chart's logo on Artifact Hub. |
| **`dependencies`**| No | A list of other charts (subcharts) required by this chart (e.g., an internal database). |
| **`kubeVersion`** | No | SemVer constraint of supported Kubernetes versions (e.g., `>= 1.26.0-0`). |
| **`deprecated`**  | No | Boolean (`true`/`false`) marking whether the chart is no longer maintained. |

### Critical CKA Exam Distinction: `version` vs `appVersion`

| Field | Meaning | When Does It Change? | Example |
| :--- | :--- | :--- | :--- |
| **`version`** | Version of the **Helm Chart package** | When you edit templates, values, or dependencies | `1.2.0` |
| **`appVersion`** | Version of the **application/container** | When you upgrade the app software (e.g., nginx/WordPress) | `2.5.1` |

> **CKA Rule of Thumb:** In Helm 3, **`apiVersion: v2`**, **`name`**, and **`version`** are the only strictly mandatory fields in `Chart.yaml`.

---

## 3. Chart Types: `application` vs `library`

Inside `Chart.yaml`, the `type` field can be:
- **`application` (default):** A standard, deployable chart that creates running Kubernetes resources (Deployments, Services, etc.).
- **`library`:** A helper chart that provides shared template utilities for other charts. **Cannot** be deployed on its own with `helm install`.

---

## 4. `templates/` and `values.yaml`

Helm combines the templates and the values file to render final Kubernetes manifests:

```text
[templates/deployment.yaml]  +  [values.yaml]  ===>  Rendered Kubernetes YAML
```

### Example Template (`templates/deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-web
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: web
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### Example Values (`values.yaml`):
```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
```

---

## 5. Essential Chart Commands for CKA

| Command | What It Does |
| :--- | :--- |
| `helm create <chart-name>` | Generates a new chart with standard boilerplate structure |
| `helm lint <path-to-chart>` | Validates chart syntax and structure for errors |
| `helm template <release-name> <path-to-chart>` | Renders chart templates locally **without** contacting Kubernetes API (great for testing) |
| `helm package <path-to-chart>` | Packages a chart directory into a `.tgz` archive |
