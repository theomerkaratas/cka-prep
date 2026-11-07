# Customizing Chart Parameters

- **Official Documentation:** [Helm Values Files](https://helm.sh/docs/chart_template_guide/values_files/)

When deploying a chart, you almost always need to customize parameters (e.g., replicas, service types, passwords, resource limits) rather than using the defaults.

Helm merges your custom inputs on top of the chart's default `values.yaml`.

---

## 1. Inspecting Default Parameters First

Before overriding values, find out what configuration keys exist in the chart:

```bash
# View all default values of a chart
helm show values bitnami/nginx

# Filter for specific keys using grep (CKA time-saver)
helm show values bitnami/nginx | grep -E "replicaCount|service" -A 5
```

---

## 2. Three Ways to Customize Parameters

### Method 1: Using the `--set` Flag (Fastest for CKA)

Best for quick, one-off overrides during the exam when only 1 or 2 fields need to change.

```bash
# Basic single value override
helm install my-web bitnami/nginx --set replicaCount=3

# Overriding multiple values (comma-separated or multiple --set flags)
helm install my-web bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort

# Overriding nested values
helm install my-web bitnami/nginx --set image.tag="1.25-alpine"

# Setting an array/list item
helm install my-web bitnami/nginx --set ingress.hosts[0]=example.com
```

> **Useful `--set` variants:**
>
> - `--set-string`: Forces the value to be parsed as a string (e.g., `--set-string version="1.0"` so `1.0` isn't treated as a float).
> - `--set-file`: Injects the content of a local file as a value (e.g., `--set-file sslCert=./cert.pem`).

---

### Method 2: Using a Custom Values File (`-f` or `--values`)

Best practice when you have multiple complex configurations. You do not need to copy the entire `values.yaml`; **only specify the keys you want to override**:

**1. Create a custom values file (`custom-values.yaml`):**

```yaml
replicaCount: 3

service:
  type: NodePort
  port: 8080

resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

**2. Pass the file to `helm install`:**

```bash
helm install my-web bitnami/nginx -f custom-values.yaml
```

> **Tip:** You can pass multiple `-f` files. The rightmost file overrides earlier files:
>
> ```bash
> helm install my-web bitnami/nginx -f base-values.yaml -f prod-values.yaml
> ```
>
> This allows keeping shared settings in `base-values.yaml` while overriding only environment-specific changes (e.g., prod replicas) in `prod-values.yaml`.

---

### Method 3: Pulling and Editing the Chart Locally (`helm pull --untar`)

If you need to inspect or modify the chart manifests and `values.yaml` directly on the local filesystem:

```bash
# 1. Download and unpack the chart archive into a local folder
helm pull bitnami/nginx --untar

# 2. Inspect/edit files directly
ls -la nginx/
vi nginx/values.yaml

# 3. Install directly from the local directory path
helm install my-web ./nginx
```

---

## 3. Order of Precedence (Which Value Wins?)

When multiple sources provide the same configuration key, Helm merges them according to strict priority:

```text
┌────────────────────────────────────────────────┐
│  1. --set flags               (HIGHEST)        │
│  2. -f / --values files       (Rightmost wins) │
│  3. Default values.yaml       (LOWEST)         │
└────────────────────────────────────────────────┘
```

**Example:**
If `values.yaml` sets `replicaCount: 1`, `custom-values.yaml` sets `replicaCount: 3`, and you run:

```bash
helm install my-web bitnami/nginx -f custom-values.yaml --set replicaCount=5
```

The final replica count in the cluster will be **`5`** because `--set` has the highest priority.

---

## 4. Checking Currently Applied Values

To see which values are actively applied to an existing release:

```bash
# Show only user-supplied custom overrides
helm get values my-web

# Show ALL computed values (defaults merged with user overrides)
helm get values my-web --all
```

---

## 5. CKA Exam Summary: What You Actually Need

In the CKA exam, customizing chart parameters boils down to just these tools:

- **`--set key=value` (~90% of exam scenarios):** Used when the question asks you to deploy or modify a chart with 1–2 specific overrides (e.g., set replica count, service type, or image tag).
- **`-f / --values file.yaml`:** Used when the question provides a ready-made YAML file and asks you to apply it during installation.
- **`helm show values <chart>`:** Used to quickly look up exact parameter key names if you are unsure how the chart structured them.
- **`helm get values <release>`:** Used to verify what custom overrides are currently active on a running release.
