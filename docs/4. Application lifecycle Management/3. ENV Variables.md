# Environment Variables

Kubernetes provides multiple flexible ways to inject environment variables into container runtimes.

---

## 1. Direct Key-Value Pairs

The simplest method is defining literal values directly inside the container specification:

```yaml
# pod-plain-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-plain-env
spec:
  containers:
  - name: myapp
    image: nginx
    env:
    - name: APP_COLOR
      value: "blue"
    - name: APP_MODE
      value: "production"
```

---

## 2. Using `valueFrom` (Individual Keys from ConfigMaps/Secrets)

When referencing a specific key from an existing ConfigMap or Secret:

```yaml
# pod-value-from.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-value-from
spec:
  containers:
  - name: myapp
    image: nginx
    env:
    # Read single key from a ConfigMap
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: color

    # Read single key from a Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password

    # Read Pod metadata using Downward API (fieldRef)
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
```

---

## 3. Using `envFrom` (Bulk Injection of All Keys)

If you want to inject **all key-value pairs** from a ConfigMap or Secret as environment variables in one declaration:

```yaml
# pod-env-from.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-env-from
spec:
  containers:
  - name: myapp
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-credentials
```

> **Note:** When using `envFrom`, every key in the ConfigMap or Secret must follow valid environment variable naming conventions (`[A-Za-z_][A-Za-z0-9_]*`), otherwise invalid keys are skipped.
