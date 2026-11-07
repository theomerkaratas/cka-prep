# ConfigMaps

A **ConfigMap** is an API object used to store non-confidential configuration data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a Volume.

ConfigMaps are **namespaced** resources.

---

## Creating ConfigMaps

### Imperative Commands (CKA Recommended)

```bash
# From literal key-value pairs
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=ENVIRONMENT=production \
  --from-literal=LOG_LEVEL=INFO

# From an entire configuration file (key becomes the filename)
kubectl create configmap app-config --from-file=app.properties

# From a directory containing multiple config files
kubectl create configmap app-config --from-file=/path/to/configs/

# From an environment-variable syntax file
kubectl create configmap app-config --from-env-file=.env
```

### Declarative Manifest

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  APP_COLOR: blue
  ENVIRONMENT: production
  LOG_LEVEL: INFO
```

---

## Consuming ConfigMaps in Pods

### 1. Individual Environment Variables (`env` + `configMapKeyRef`)
```yaml
spec:
  containers:
  - name: myapp
    image: nginx
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_COLOR
```

### 2. Bulk Environment Variables (`envFrom` + `configMapRef`)
```yaml
spec:
  containers:
  - name: myapp
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config
```

### 3. Mounted as Files in a Volume
Each key in the ConfigMap becomes an individual file inside the mounted directory, with the value as the file contents:

```yaml
# pod-volume-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-vol
spec:
  containers:
  - name: myapp
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config # Directory where config files appear
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```
