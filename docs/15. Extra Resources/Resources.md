https://kubernetes.io/

### Scheduler
- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduling_code_hierarchy_overview.md
- https://kubernetes.io/blog/2017/03/advanced-scheduling-in-kubernetes
- https://jvns.ca/blog/2017/07/27/how-does-the-kubernetes-scheduler-work
- https://stackoverflow.com/questions/28857993/how-does-kubernetes-scheduler-work

### Secrets
Secret Store CSI Driver -> https://www.youtube.com/watch?v=MTnQW9MxnRI

> How to encrypt on `etcd` level -> https://learn.kodekloud.com/learn/courses/cka-certification-course-certified-kubernetes-administrator/module/2ddcf79b-abb0-4aeb-ad0c-3d54c7b4fc64/lesson/4abaab5a-b18a-45a4-b969-2f2251a54375
> 
> https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/ -> better encrytion

> most important thing is execute missing commands like `kubectl create secret`. Output will show you the options

### App Lifecycle Management
Init Containers -> https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
Sidecar Containers -> https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/

# Cluster Upgrade
`kubeadm upgrade` -> https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/
how to upgrade with `kubeadm` -> https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

## Backup 
Mastering etcd Backup in Kubernetes -> https://medium.com/@Ibraheemcisse/mastering-etcd-backup-in-kubernetes-cka-ready-guide-55363642a60c

CKA tag medium -> https://medium.com/tag/cka

`etcd` -> https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
`etcd` recovery -> https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md


> Note: We won't know if what we did is correct or not. So we must verify our work ourselves. For example, if the question is to create a pod with a specific image, we must run the `kubectl describe pod` command

## Storage
PV -> https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes\
