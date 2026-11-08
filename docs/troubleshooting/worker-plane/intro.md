# Worker Plane Failure: Overview (CKA)

Worker node failures are part of the Troubleshooting domain (30%). A worker node that is `NotReady` or `Unknown` stops running pods. Start with the node status, then check the node's resources, its kubelet, and the kubelet certificates.

## Troubleshooting Order

1. [Node Status](node-status.md): is the node `Ready`, and what do its conditions say?
2. [Unknown Node](node-unknown.md): the node stopped reporting. Is it crashed or offline?
3. [Resources](resources.md): CPU, memory, disk, and process count.
4. [kubelet](kubelet.md): is the service running, and what do its logs say?
5. [API Server Endpoint](api-server-endpoint.md): does the kubelet point to a reachable control-plane endpoint?
6. [kubelet Certificates](kubelet-certificates.md): are they valid, in the right group, and signed by the right CA?
7. [Exam Workflow](exam-workflow.md): steps and tips for the exam.
