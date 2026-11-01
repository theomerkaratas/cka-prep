# Scheduler Profiles & Extension Points

Starting with Kubernetes v1.18+, the scheduler is built on a modular plugin architecture known as the **Scheduling Framework**. 

Instead of running separate scheduler binaries for different scheduling behaviors, you can define **multiple scheduler profiles** within a single `kube-scheduler` process.

---

## Scheduling Stages and Extension Points

The scheduling process is split into two phases: the **Scheduling Cycle** (runs for one pod at a time, selects node) and the **Binding Cycle** (binds pod to node, runs concurrently).

```text
1. Scheduling Queue
   └── queueSort                -> PrioritySort

2. Filtering (Nodes that cannot run the pod are eliminated)
   ├── preFilter
   ├── filter                   -> NodeResourcesFit, NodeName, NodeUnschedulable,
   │                               TaintToleration, NodePorts, NodeAffinity
   └── postFilter

3. Scoring (Remaining nodes are ranked 0–100)
   ├── preScore
   ├── score                    -> NodeResourcesFit, ImageLocality, TaintToleration,
   │                               NodeAffinity
   ├── normalizeScore
   └── reserve

4. Binding (Pod is assigned and bound to chosen node)
   ├── permit                   -> Approves, delays, or denies binding
   ├── preBind
   ├── bind                     -> DefaultBinder (writes binding to API server)
   └── postBind
```

---

## Configuring Multiple Profiles in One Scheduler

By defining multiple profiles in `KubeSchedulerConfiguration`, each profile acts as an independent virtual scheduler with its own custom plugins enabled or disabled:

```yaml
# scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: false
profiles:
  # Profile 1: Custom plugins for specific scoring
  - schedulerName: my-custom-scheduler-1
    plugins:
      score:
        disabled:
        - name: TaintToleration
        enabled:
        - name: MyCustomPluginA
        - name: MyCustomPluginB

  # Profile 2: Disables all default scoring plugins
  - schedulerName: my-custom-scheduler-2
    plugins:
      preScore:
        disabled:
        - name: "*"
      score:
        disabled:
        - name: "*"

  # Profile 3: Default behavior with all standard plugins
  - schedulerName: my-custom-scheduler-3
```

Pod manifests can target any of these virtual profiles simply by specifying `spec.schedulerName: my-custom-scheduler-1`.
