[Kubernetes Documentation](https://kubernetes.io/)

### Scheduler
- [Scheduling code hierarchy](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduling_code_hierarchy_overview.md)
- [Advanced scheduling in Kubernetes](https://kubernetes.io/blog/2017/03/advanced-scheduling-in-kubernetes)
- [How does the Kubernetes scheduler work?](https://jvns.ca/blog/2017/07/27/how-does-the-kubernetes-scheduler-work)
- [How does the Kubernetes scheduler work? (Stack Overflow)](https://stackoverflow.com/questions/28857993/how-does-kubernetes-scheduler-work)

### Secrets
Secret Store CSI Driver -> [video](https://www.youtube.com/watch?v=MTnQW9MxnRI)

> How to encrypt on `etcd` level -> [KodeKloud lesson](https://learn.kodekloud.com/learn/courses/cka-certification-course-certified-kubernetes-administrator/module/2ddcf79b-abb0-4aeb-ad0c-3d54c7b4fc64/lesson/4abaab5a-b18a-45a4-b969-2f2251a54375)
> 
> [Encrypt data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) -> better encrytion

> most important thing is execute missing commands like `kubectl create secret`. Output will show you the options

### App Lifecycle Management
Init Containers -> [Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
Sidecar Containers -> [Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)

# Cluster Upgrade
`kubeadm upgrade` -> [command reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/)
how to upgrade with `kubeadm` -> [upgrade guide](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

## Backup 
Mastering etcd Backup in Kubernetes -> [Medium article](https://medium.com/@Ibraheemcisse/mastering-etcd-backup-in-kubernetes-cka-ready-guide-55363642a60c)

CKA tag medium -> [Medium CKA tag](https://medium.com/tag/cka)

`etcd` -> [backup guide](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
`etcd` recovery -> [recovery guide](https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md)


> Note: We won't know if what we did is correct or not. So we must verify our work ourselves. For example, if the question is to create a pod with a specific image, we must run the `kubectl describe pod` command

## Storage
PV -> [Persistent Volumes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes)
