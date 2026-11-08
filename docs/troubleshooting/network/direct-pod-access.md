# Step 2: Reach the Pods Directly

This step skips the Service. It tests whether the app answers on the pod IP and port. If it works here, the problem is more likely in the Service layer.

## Step 2.1: Get the Pod IPs

```bash
# Pod IPs, one line per pod
kubectl get pods -l app=hostnames -o wide

# Only the IP addresses
kubectl get pods -l app=hostnames -o jsonpath='{.items[*].status.podIP}'
```

## Step 2.2: Find the Container Port

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].ports[*].containerPort}'
```

## Step 2.3: Start a Temporary Test Pod

```bash
kubectl run busybox --rm -it --restart=Never --image=busybox:1.28 -- sh
```

You are now inside the `busybox` pod. Type `exit` to leave it and delete it.

## Step 2.4: Call the Pod IP

Inside the `busybox` pod, call each pod IP and port:

```bash
wget -qO- -T 2 http://<pod-ip>:<port>
```

Repeat for each IP. You should get a response from each pod.

## What to Do Next

- **Every pod answers:** the app works. Go to [Step 3](service-endpoints.md).
- **A pod does not answer:** the problem is in the app or the container, not the Service. Check the app logs and the container port.
