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

1. **[Core Components](core-components/etcd.md):** `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kubelet`, `kube-proxy`, and essential CLI commands.
2. **[Scheduling](scheduling/pod-placement/manuel-scheduling.md):** Manual scheduling, labels & selectors, taints & tolerations, node affinity, static pods, and multiple schedulers.
3. **[Logging & Monitoring](monitoring/cluster-monitoring/monitoring-cluster-components.md):** Metrics Server (`cAdvisor`), `kubectl top`, and container log management.
4. **[Application Lifecycle Management](alm/workload-lifecycle/rolling-updates-and-rollbacks.md):** Rolling updates, rollbacks, commands/args, ConfigMaps, Secrets, init containers, and autoscaling (HPA/VPA).
5. **[Cluster Maintenance](maintenance/os-upgrade.md):** Node drain/cordon, OS upgrades, cluster version upgrades with `kubeadm`, and `etcd` backup & restore.
6. **[Security](security/identity-and-access-control/security-primitives.md):** Authentication, TLS/PKI, Certificates API, Kubeconfig, Authorization (Node, RBAC), ServiceAccounts, SecurityContexts, and NetworkPolicies.
7. **[Storage](storage/storage-fundamentals/intro.md):** Storage drivers, CSI, Volumes, PVs, PVCs, Reclaim policies, and StorageClasses.
8. **[Networking](networking/foundations/network-layers.md):** Switching/routing, Linux network namespaces, CNI addons, IPAM, Service networking, CoreDNS, Ingress, and Gateway API.
9. **[Cluster Design](cluster-design/configuring-ha.md):** High availability planning, infrastructure sizing, and multi-master etcd.
10. **[Helm](helm/intro.md):** Package management, chart customization, and lifecycle releases.
11. **[Kustomize](kustomize/introduction/intro.md):** Declarative template-free configuration management, overlays, and transformers.
12. **[Troubleshooting](troubleshooting/intro.md):** Diagnostic methodology for control plane, workers, applications, and networks.
---

## Key Exam Tips & Reflexes

- **Speed via Aliases:** Set these up at the start of your exam session:

  ```bash
  alias k=kubectl
  export do="--dry-run=client -o yaml"
  ```
- **Kubernetes Documentation:** [Kubernetes Documentation](https://kubernetes.io/)
- **More CKA articles:** [Medium CKA tag](https://medium.com/tag/cka)
- **Documentation Bookmarks:** Get familiar with standard YAML examples on [kubernetes.io/docs](https://kubernetes.io/docs/).
- **Explain Over Search:** When in doubt about manifest structure, use `kubectl explain <resource>.<field>` instead of browsing the web.
