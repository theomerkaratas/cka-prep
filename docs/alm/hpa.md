manuel way:
```sh
kubectl scale deployment deploy-name --replicas=3
```
we can do HPA imperatively:
```sh
kubectl autoscale deployment deploy-name --cpu='50%' --min=1 --max=10
```
So if CPU usage is beyond 50%, HPA modifies the number of replicas depending on the usage

We can list HPAs with:
```sh
kubectl get hpa
```
- HPA is not namespace level
Declarative way:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

We can use HPA for ReplicaSets, Deployments and StatefulSets