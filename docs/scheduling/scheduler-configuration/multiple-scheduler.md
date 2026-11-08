# Multiple Schedulers

Kubernetes allows you to deploy and run multiple schedulers alongside the default scheduler (`default-scheduler`). Different pods can then specify which scheduler should handle their placement.

---

## 1. Scheduler Configuration File

A custom scheduler is configured using a `KubeSchedulerConfiguration` manifest:

```yaml
# my-scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-custom-scheduler
leaderElection:
  leaderElect: true
  resourceNamespace: kube-system
  resourceName: lock-object-my-custom-scheduler # Must be unique! Do not conflict with default kube-scheduler
```

> **Important on Leader Election:** If running multiple replicas or running alongside the default scheduler, ensure `resourceName` is unique. If it matches `kube-scheduler`, both schedulers will compete for the same leader lock.

---

## 2. Deploying the Custom Scheduler

You can deploy the custom scheduler as a Pod or Deployment in the `kube-system` namespace:

```yaml
# my-custom-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  serviceAccountName: my-scheduler-sa
  containers:
  - name: kube-scheduler
    image: registry.k8s.io/kube-scheduler:v1.30.0
    command:
    - kube-scheduler
    - --config=/etc/kubernetes/my-scheduler/my-scheduler-config.yaml
    - --v=2
    volumeMounts:
    - name: config-volume
      mountPath: /etc/kubernetes/my-scheduler
  volumes:
  - name: config-volume
    configMap:
      name: my-scheduler-config
```

---

## 3. Directing Pods to Use the Custom Scheduler

To assign a pod to the custom scheduler, set `spec.schedulerName`:

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom
spec:
  schedulerName: my-custom-scheduler
  containers:
  - name: nginx
    image: nginx:alpine
```

> If the designated scheduler is misconfigured or not running, the pod will remain stuck in `Pending` state.

---

## 4. Verification and Troubleshooting

### Check Scheduler Logs
```bash
kubectl logs -n kube-system my-custom-scheduler
```

### Inspect Scheduling Events
```bash
kubectl get events -o wide
```

Sample event output showing the custom scheduler successfully assigning the node:
```text
LAST SEEN   COUNT   NAME           KIND   TYPE     REASON      SOURCE                MESSAGE
9s          1       nginx-custom   Pod    Normal   Scheduled   my-custom-scheduler   Successfully assigned default/nginx-custom to node01
8s          1       nginx-custom   Pod    Normal   Pulling     kubelet, node01       pulling image "nginx:alpine"
```

