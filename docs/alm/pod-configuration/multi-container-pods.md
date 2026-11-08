- they share the same network namespace, so they can refer to each other as localhost
- they have access to the same storage volumes
### Design Patterns
```text
Co-located Containers           Regular Init Containers          Sidecar Containers
[container A] <-> [container B] [init] --(done)--> [main]       [main] <-> [sidecar]
(started together, no order)    (init runs before main)          (helper runs beside main)
```

> Main difference between co-located and sidecar is, with co-located there is no ability to define which container starts first and the order of that startup.

### Co-located Containers
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    type: webapp
spec:
  containers:
  - name: web-app
    image: web-app
    ports:
    - containerPort: 8080
  - name: main-app
    image: main-app
```
### Init Containers
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    type: webapp
spec:
  containers:
  - name: web-app
    image: web-app
    ports:
    - containerPort: 8080
  initContainers:
  - name: db-checker
    image: busybox
    command: ['wait-for-db-to-start.sh']
```
### Sidecar Containers
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    type: webapp
spec:
  containers:
  - name: web-app
    image: web-app
    ports:
    - containerPort: 8080
  initContainers:
  - name: log-shipper
    image: busybox
    command: ['setup-log-shipper.sh']
    restartPolicy: Always # this has to be always, thats why sidecar
```

## Further Reading
- [Sidecar Containers documentation](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
  