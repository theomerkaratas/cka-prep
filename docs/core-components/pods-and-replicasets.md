# Pods and ReplicaSets

This note covers the foundational workload primitives in Kubernetes: Pods, ReplicationControllers, and ReplicaSets.

---

## 1. Pods

A **Pod** is the smallest execution unit in Kubernetes. It encapsulates one or more containers that share storage, network IP, and specifications on how to run.

### Creating a Pod Imperatively
```bash
kubectl run nginx --image=nginx
```

### Declarative Pod Manifest
```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Apply and inspect:
```bash
# Apply manifest
kubectl apply -f pod.yaml

# List pods in a specific namespace
kubectl get pods -n <namespace_name>

# Get detailed information
kubectl describe pod mypod
```

---

## 2. ReplicationController (Legacy)

A `ReplicationController` ensures that a specified number of pod replicas are running at any given time. It is the older, legacy mechanism superseded by `ReplicaSet`.

```yaml
# rc-def.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
    tier: frontend
spec:
  replicas: 3
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
```

Apply and verify:
```bash
kubectl apply -f rc-def.yaml
kubectl get replicationcontroller
```

---

## 3. ReplicaSets

A **ReplicaSet** is the modern replacement for ReplicationController. The key difference is that ReplicaSets support flexible set-based selectors (`matchLabels` and `matchExpressions`).

> **Note:** While ReplicaSets can be used independently, in production and on the CKA exam you will almost always manage them through **Deployments**.

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
  labels:
    app: myapp
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        app: myapp
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
```

### Scaling a ReplicaSet

You can scale ReplicaSets declaratively or imperatively:

```bash
# Imperatively via command
kubectl scale replicaset myapp-rs --replicas=5

# Scaling using manifest file
kubectl scale --replicas=6 -f replicaset.yaml

# Or edit the running ReplicaSet directly
kubectl edit replicaset myapp-rs
```

