manual way:
```sh
kubectl edit deployment deployment-name
```
> Unlike the HPA, the VPA does not come built-in. We must deploy it.

```sh
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml
```
VPA deployment consists of multiple components.

```text
$ kubectl get pods -n kube-system | grep vpa
vpa-admission-controller-xxxx   Running
vpa-recommender-xxxx            Running
vpa-updater-xxxx                Running
```
# How to Create

Declarative way:
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto" # "Recreate", "Initial" or "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: "my-app"
      minAllowed:
        cpu: "250m"
      maxAllowed:
        cpu: "2"
      controlledResources: ["cpu"]
```
> There is no imperative command, because it is not built-in component like HPA

> We need at least 2 replica for healthy VPA process

# How VPA Works (Not need for CKA)

**VPA Recommender:** 
- Continuously monitoring the resource usage from the K8S Metrics API 
- Collecting historical and live usage data for pods 
- Provides recommendations on optimal CPU and mem values. 
- It does not modify the pod directly; it only suggest changes
**VPA Updater:** 
- Detects pods that are running with suboptimal resources and evicts them when an update is needed
- Gets the info from recommender and monitors the pod and if a pod needs to updated, it evicts it
**VPA Admission Controller:**
- Intervenes in the pod creation process and uses the recommendations from the recommender again to mutate the pod spec to apply the recommended CPU and mem values at startup
- this ensures that the newly created pods start with the correct resource requests

> Basically VPA recommender collects info, the updater monitors or get the info from the recommender, compares that to the actual Pod. And if the pod is beyond the threshold, it kills the pod.