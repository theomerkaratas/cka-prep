# Network Failure: Exam Workflow and Checklist

Network problems are part of the Troubleshooting domain (30%). When an application cannot be reached, or pods cannot talk to each other, check the network in a fixed order. Each step rules out one layer before you move to the next.

## Key Components

| Component | What it does | If it breaks |
| :--- | :--- | :--- |
| **Pod** | Runs the app. Each pod gets its own IP. | App is down or restarting |
| **Service** | Gives a stable address for a group of pods | Traffic does not reach any pod |
| **CoreDNS** | Turns service names into IP addresses | Names do not resolve |
| **CNI plugin** | Gives pods IP addresses and sets up pod-to-pod traffic | Pods stay `Pending`, or cannot talk |
| **kube-proxy** | Writes the rules that send service traffic to pods | Service IP works in DNS but not in traffic |

## Troubleshooting Order

Follow these steps in order. Start at the top and stop at the first step that fails.

1. [Step 1: Check the Pods](pods.md): are the pods `Running` and `Ready`?
2. [Step 2: Reach the Pods Directly](direct-pod-access.md): does the app answer on the pod IP?
3. [Step 3: Check the Service and Endpoints](service-endpoints.md): does the Service select the pods?
4. [Step 4: Check CoreDNS](coredns.md): are the DNS pods healthy?
5. [Step 5: Check DNS Resolution](dns-resolution.md): do pods resolve names correctly?
6. [Step 6: Check the CNI Plugin](cni.md): do the network plugin pods run?
7. [Step 7: Check kube-proxy](kube-proxy.md): are the service rules on the node?

When you are done, use the [Exam Workflow](exam-workflow.md) as a checklist.

## Test Setup Used in These Steps

The examples use a Deployment named `hostnames` with the label `app=hostnames` and a Service named `hostnames` in the `default` namespace. Replace the names with your own.
