# Certified Kubernetes Administrator (CKA) Study Guide

Welcome to the comprehensive study and preparation guide for the **Certified Kubernetes Administrator (CKA)** examination by the Cloud Native Computing Foundation (CNCF) and Linux Foundation.

---

## CKA Exam Curriculum Overview

The Certified Kubernetes Administrator (CKA) exam tests practical, performance-based administration skills across five core domains:

| Domain | Weight | Core Focus Areas |
| :--- | :---: | :--- |
| **Storage** | **10%** | StorageClasses, PersistentVolumes (PV), PersistentVolumeClaims (PVC), Volume Mounts, CSI. |
| **Troubleshooting** | **30%** | Application failures, Node failures, Control plane crashes, Networking and CNI issues. |
| **Workloads & Scheduling** | **15%** | Deployments, Rolling Updates, DaemonSets, Resource Requests/Limits, Affinity/Taints, PriorityClasses. |
| **Cluster Architecture, Installation & Config** | **25%** | Kubeadm upgrades, ETCD backup & restore, Static Pods, RBAC (Roles & RoleBindings), Kubeconfig. |
| **Services & Networking** | **20%** | Services (ClusterIP, NodePort), Ingress, NetworkPolicies, CoreDNS, CNI and IPAM. |

---

## Structure of This Documentation

This knowledge base is organized sequentially into structured, modular topics:

1. **[Core Components](1.%20Core%20Components/1.%20etcd.md):** `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kubelet`, `kube-proxy`, and essential CLI commands.
2. **[Scheduling](2.%20Scheduling/1.%20Manuel%20Scheduling.md):** Manual scheduling, labels & selectors, taints & tolerations, node affinity, static pods, and multiple schedulers.
3. **[Logging & Monitoring](3.%20Logging%20&%20Monitoring/1.%20Monitoring%20Cluster%20Components.md):** Metrics Server (`cAdvisor`), `kubectl top`, and container log management.
4. **[Application Lifecycle Management](4.%20Application%20lifecycle%20Management/1.%20Rolling%20Updates%20and%20Rollbacks.md):** Rolling updates, rollbacks, commands/args, ConfigMaps, Secrets, init containers, and autoscaling (HPA/VPA).
5. **[Cluster Maintenance](5.%20Cluster%20Maintenance/1.%20OS%20Upgrade.md):** Node drain/cordon, OS upgrades, cluster version upgrades with `kubeadm`, and `etcd` backup & restore.
6. **[Security](6.%20Security/1.%20Security%20Primitives.md):** Authentication, TLS/PKI, Certificates API, Kubeconfig, Authorization (Node, RBAC), ServiceAccounts, SecurityContexts, and NetworkPolicies.
7. **[Storage](7.%20Storage/1.1.%20Intro.md):** Storage drivers, CSI, Volumes, PVs, PVCs, Reclaim policies, and StorageClasses.
8. **[Networking](8.%20Networking/0.%20Network%20Layers.md):** Switching/routing, Linux network namespaces, CNI addons, IPAM, Service networking, CoreDNS, Ingress, and Gateway API.
9. **[Design & Install](9.%20Desing%20and%20Install%20K8S Cluster/1.%20Designing%20a%20K8S%20Cluster.md):** High availability planning, infrastructure sizing, and multi-master etcd.
10. **[Install with kubeadm](10.%20Install%20with%20kubeadm/1.%20Intro.md):** Bootstrap clusters and local Vagrant testing environments.
11. **[Helm](11.%20Helm/1.%20Intro.md):** Package management, chart customization, and lifecycle releases.
12. **[Kustomize](12.%20kustomize/1.%20Intro.md):** Declarative template-free configuration management, overlays, and transformers.
13. **[Troubleshooting](13.%20Troubleshooting/1.%20Intro.md):** Diagnostic methodology for control plane, workers, applications, and networks.

---

## Key Exam Tips & Reflexes

- **Speed via Aliases:** Always set up `alias k=kubectl` and export `do="--dry-run=client -o yaml"` at the start of your exam session.
- **Documentation Bookmarks:** Get familiar with standard YAML examples on [kubernetes.io/docs](https://kubernetes.io/docs/).
- **Explain Over Search:** When in doubt about manifest structure, use `kubectl explain <resource>.<field>` instead of browsing the web.
