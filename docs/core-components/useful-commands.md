# Useful Commands & CKA Tips

A collection of high-frequency `kubectl` imperative commands and tips essential for speed and accuracy in the CKA exam.

---

## 1. Quick Aliases & Autocompletion

Set these up immediately in exam environments to save typing time:

```bash
# Enable bash completion and alias
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k

# Fast dry-run shorthand
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"
```

---

## 2. Workload Creation & Dry-Run Generation

```bash
# Run a pod
kubectl run nginx --image=nginx

# Generate Pod YAML template
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Create a Deployment
kubectl create deployment my-deploy --image=nginx --replicas=3

# Generate Deployment YAML template
kubectl create deployment my-deploy --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml

# Create a Pod and simultaneously expose it via a Service
kubectl run httpd --image=httpd:alpine --port=80 --expose
```

---

## 3. Exposing Services

```bash
# Expose a Deployment on port 80 (ClusterIP)
kubectl expose deployment my-deploy --port=80 --target-port=80

# Expose as a NodePort
kubectl expose deployment my-deploy --type=NodePort --port=80 --target-port=80

# Expose a Pod as a Service
kubectl expose pod nginx --port=80 --name=nginx-svc
```

---

## 4. Updates, Scaling, and Editing

```bash
# Scale a deployment
kubectl scale deployment my-deploy --replicas=5

# Update container image in deployment
kubectl set image deployment/my-deploy nginx=nginx:1.25

# Edit resource directly in default editor
kubectl edit deployment my-deploy

# Delete a pod immediately without waiting 30 seconds
kubectl delete pod mypod --force --grace-period=0
```

---

## 5. API Exploration with `api-resources` & `explain`

When you forget the structure or API version of a manifest, use `explain` rather than searching documentation:

```bash
# List all API resources, shortnames, and their API groups
kubectl api-resources

# Check API version and fields of a resource
kubectl explain pod

# Inspect specific section (e.g. spec or containers)
kubectl explain pod.spec
kubectl explain pod.spec.containers

# View all nested fields recursively
kubectl explain pod.spec.tolerations --recursive
```

---

## 6. Verifying Resources

After creating or changing a resource, verify that it matches the requirements. For example, inspect a Pod's image and status with:

```bash
kubectl describe pod <pod-name>
```
