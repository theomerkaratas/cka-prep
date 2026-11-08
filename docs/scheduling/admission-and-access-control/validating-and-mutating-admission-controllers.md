# Validating and Mutating Admission Controllers

Admission controllers can be built-in or external. When external, Kubernetes calls HTTP callbacks (webhooks) using:
- **Mutating Admission Webhooks**
- **Validating Admission Webhooks**

---

## Mutating vs. Validating

| Type | Role | Example |
| :--- | :--- | :--- |
| **Mutating** | Modifies or injects fields into the resource object before admission. | Injects Istio sidecar proxy, adds default labels, sets storage class. |
| **Validating** | Inspects the resource and admits or rejects the request. | Ensures images come only from approved registries, enforces naming rules. |

### Execution Order

```text
Incoming API Request
       │
       ▼
[ Mutating Admission Controllers / Webhooks ]   <-- 1. Modifies the object
       │
       ▼
[ Object Schema Validation ]                   <-- 2. Checks schema validity
       │
       ▼
[ Validating Admission Controllers / Webhooks ] <-- 3. Confirms or rejects
       │
       ▼
Persisted to etcd
```

> **Why this order?** Mutating controllers run first so that any modifications they make can be fully checked by the schema validator and validating admission controllers.

---

## 1. Webhook Configuration Manifest

A `MutatingWebhookConfiguration` instructs the API server which requests should be sent to your external webhook server:

```yaml
# mutating-webhook-configuration.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-mutation-webhook
webhooks:
- name: webhook-server.my-webhook.svc
  clientConfig:
    service:
      name: webhook-server
      namespace: webhook-demo
      path: "/mutate"
      port: 443
    caBundle: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0t..." # Base64-encoded CA cert of webhook server
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
```

---

## 2. Webhook Service Definition

The webhook server must be exposed as a Service (typically on port `443` over TLS):

```yaml
# webhook-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook-server
  namespace: webhook-demo
spec:
  selector:
    app: webhook-server
  ports:
  - port: 443
    targetPort: 8443
```

---

## 3. Webhook Server Deployment

The webhook server itself runs as a standard Kubernetes Deployment, with TLS certificates mounted:

```yaml
# webhook-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook-server
  namespace: webhook-demo
  labels:
    app: webhook-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webhook-server
  template:
    metadata:
      labels:
        app: webhook-server
    spec:
      containers:
      - name: server
        image: my-webhook-image:latest
        ports:
        - containerPort: 8443
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/webhook/certs
          readOnly: true
      volumes:
      - name: tls-certs
        secret:
          secretName: webhook-server-tls
```
