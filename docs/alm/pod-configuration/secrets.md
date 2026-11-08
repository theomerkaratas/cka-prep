- Similar to ConfigMaps, but they are stored in an encoded or hashed format.
- Namespace level
## How to Create Secrets
We can create secrets with this command below
```sh
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secret123
```
or
```sh
kubectl create secret generic db-credentials --from-file=app_secret.properties
```
then
```properties
username=admin
password=secret123
```
there are some other options like:
- generic -> opaque (most common way)
- `TLS`
- `docker registery`
- `basic-auth`
- `ssh-auth`
- custom
also we can create with declarative way:
```yaml
apiversion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=      # base64 encoded "admin"
  password: c2VjcmV0MTIz= # base64 encoded "secret123"
```
or
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: secret123
```
if we use string data way, we do not need to encode it

## How to encode and decode
```sh
echo -n "admin" | base64
# Output: YWRtaW4=
echo "YWRtaW4=" | base64 -d # or --decode
# Output: admin
```
## How to Inject Into Pods
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    tier: backend
spec:
  containers:
  - name: my-app
    image: my-image
    ports:
    - containerPort: 8080
    envFrom:
    - secretRef:
        name: db-credentials
```
we can inject three kind of way:
- ENV:
```yaml
envFrom:
- secretRef:
    name: db-credentials
```
- Single ENV:
```yaml
env:
- name: ADMIN
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: admin
```
- Volume
```yaml
volumes:
- name: db-credentials-valume
  configMap:
    name: db-credentials
```
> We can use `Hashicorp Vault`, `Sealed Secret` or `Helm Secrets`


>We cannot see secret values via `kubectl describe secret secret-name`, but we can see with `kubectl get secret secret-name -o yaml`

> During the exam, use commands like `kubectl create secret --help` to review available options.

## Further Reading
- Secret Store CSI Driver -> [video](https://www.youtube.com/watch?v=MTnQW9MxnRI)
- Encrypting Secrets at the `etcd` level -> [KodeKloud lesson](https://learn.kodekloud.com/learn/courses/cka-certification-course-certified-kubernetes-administrator/module/2ddcf79b-abb0-4aeb-ad0c-3d54c7b4fc64/lesson/4abaab5a-b18a-45a4-b969-2f2251a54375)
- [Encrypt data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
