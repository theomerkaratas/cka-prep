In older versions, if we change resource requirements, pods were deleted and then recreated. It was a default behavior. Then in-replace update of pods resource comes, but default behavior was not changed.
To enable this new feature, we must set `FEATURE_GATES=InPlacePodVerticalScaling=true`
One this enabled, the pod definition supports a set of `resizePolicy parameters`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
  spec:
    containers:
    - name: my-app
      image: my-image
      resizePolicy:
      - resourceName: cpu
        restartPolicy: NotRequired
      - resourceName: memory
        restartPolicy: RestartContainer
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```
# Limitations of This Feature:
- It is only work for CPU and memory
- Pod QoS class cannot change
- `initContainers` and `ephemeralContainers` cannot be resized with this approach
- resource requests and limits cannot be removed once set
- A container's memory limit cannot be set below its current usage. If a container is puts in this state, the resize status will remain inProgress until the desired memory limit becomes feasible
- Windows pods cannot be resize