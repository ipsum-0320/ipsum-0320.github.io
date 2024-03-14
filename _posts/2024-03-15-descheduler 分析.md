---
layout: mypost
title: Descheduler 分析
categories: [Descheduler]
---

> 参考文章：https://www.qikqiak.com/post/k8s-cluster-balancer/

## 1.为什么需要 Descheduler？

实例在新建时，调度器可以根据当时集群状态选择最优节点进行调度，但集群内资源使用状况是动态变化的，集群在一段时间内就会出现不均衡的状态，需要 Descheduler 将节点上已经运行的 pods 迁移到其他节点，使集群内资源分布达到一个比较均衡的状态。有以下几个原因我们希望将节点上运行的实例迁移到其他节点：

- 节点上 **pod 利用率**的变化导致某些节点利用率过低或者过高；
- 节点标签变化导致 pod 的亲和与反亲和策略不满足要求；
- 新节点上线与故障节点下线；

descheduler 会根据相关的策略挑选出节点需要迁移的实例然后删除实例，新实例会重新通过 kube-scheduler 进行调度到合适的节点上。descheduler 迁移实例的策略需要与 kube-scheduler 的策略共同使用，二者是相辅相成的。

> Descheduler 完成调度的方式如下：
>
> 1. 首先需要根据集群新的状态去确定有哪些 Pod 是需要被重新调度的。
> 2. 然后再重新找出最佳节点去调度这些 Pod。
>
> 对于第二步，`kube-scheduler` 的打分阶段就已经解决了这个问题，所以我们只需要找出被错误调度的 Pod，然后将其移除就可以。

## 2.Descheduler 的调度策略

调度策略分为两类：Balance（节点平衡）和Deschedule（pod重调度）。

| Name                                                         | Extension Point Implemented | Description                                                  |
| ------------------------------------------------------------ | --------------------------- | ------------------------------------------------------------ |
| [RemoveDuplicates](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#removeduplicates) | Balance                     | Spreads replicas 让相同workload的pod打散在不同节点           |
| [LowNodeUtilization](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#lownodeutilization) | Balance                     | Spreads pods according to pods resource requests and node resources available 平衡目标的节点的利用率 |
| [HighNodeUtilization](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#highnodeutilization) | Balance                     | Spreads pods according to pods resource requests and node resources available 让pod集中在几个节点上 |
| [RemovePodsViolatingInterPodAntiAffinity](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#removepodsviolatinginterpodantiaffinity) | Deschedule                  | Evicts pods violating pod anti affinity 驱逐pod违背了pod anti affinity |
| [RemovePodsViolatingNodeAffinity](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#removepodsviolatingnodeaffinity) | Deschedule                  | Evicts pods violating node affinity 驱逐pod违背了node affinity |
| [RemovePodsViolatingNodeTaints](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#removepodsviolatingnodetaints) | Deschedule                  | Evicts pods violating node taints 驱逐pod不能容忍节点的污点  |
| [RemovePodsViolatingTopologySpreadConstraint](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#removepodsviolatingtopologyspreadconstraint) | Balance                     | Evicts pods violating TopologySpreadConstraints 驱逐pod违背了TopologySpreadConstraints |
| [RemovePodsHavingTooManyRestarts](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#removepodshavingtoomanyrestarts) | Deschedule                  | Evicts pods having too many restarts 驱逐pod有太多次重启     |
| [PodLifeTime](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#podlifetime) | Deschedule                  | Evicts pods that have exceeded a specified age limit pod生存一段时间后执行驱逐 |
| [RemoveFailedPods](https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md#removefailedpods) | Deschedule                  | Evicts pods with certain failed reasons 驱逐处于failed的pod  |

### 1.HighNodeUtilization

介绍参考：https://midbai.com/post/descheduler-node-utilization-plugin/#highnodeutilization%E6%8F%92%E4%BB%B6%E4%BB%8B%E7%BB%8D

HighNodeUtilization它是将利用率低的节点上的pod移动到利用率高的节点，结合clusterAutoScaler将空闲的节点移出集群并进行销毁（回收），即目的是提高节点的利用率。

> 这里的利用率等同于节点request的使用率，节点request的使用率为node上所有pod的request之和占node上可以分配资源比值，即`NodeUtilization=PodsRequestsTotal * 100 / nodeAllocatable`。
>
> 低利用率 `underutilizedNodes`和高利用率节点 `overutilizedNodes`是根据thresholds进行区分，节点request的使用率大于thresholds为高利用率节点（合理利用率节点），节点request的使用率小于等于thresholds为低利用率节点。

低利用率`underutilizedNodes`和高利用率节点`overutilizedNodes`是根据thresholds进行区分，节点request的使用率大于thresholds为高利用率节点（合理利用率节点），节点request的使用率小于等于thresholds为低利用率节点。

![descheduler-nodeutilization-highNodeUtilizatio](https://midbai.com/post/descheduler-node-utilization-plugin/descheduler-nodeutilization-highNodeUtilizatio.webp)

>此外，和 `HighNodeUtilization` 策略关联的另一个参数是 `numberOfNodes`，只有当未充分利用的节点数大于该配置值的时候，才可以配置该参数来激活该策略，该参数对于大型集群非常有用，其中有一些节点可能会频繁使用或短期使用不足，默认情况下，`numberOfNodes` 为 0。

### 2.LowNodeUtilization

lowNodeUtilization插件它是将**高利用率的节点上的pod移动到低利用率的节点上**，目标是平衡节点的利用率。

这个插件将节点分为高利用率节点`overutilizedNodes`、低利用率节点`underutilizedNodes`、合理利用率节点`targetutilizedNodes`。

它是根据`thresholds` 和`targetThresholds`和`useDeviationThresholds`进行区分，这里跟highNodeUtilization不一样它有两个阈值，即低水位阈值`lowResourceThreshold`和高水位阈值`highResourceThreshold`，而`useDeviationThresholds`代表**阈值水位线是否基于所有节点的request平均利用率**。

> * 小于 `thresholds` 代表 Node 负载不足（low）。
> * 大于 `targetThresholds` 代表 Node 超载（high）。
> * 位于两者之间代表 Node 处于中间水平。

当`useDeviationThresholds`不启用的时候，`thresholds`为低水位阈值`lowResourceThreshold`，`targetThresholds`为高水位阈值`highResourceThreshold`。节点所有资源类型request的利用率都小于等于低水位阈值`lowResourceThreshold`（不包括不可调度节点）为低利用率节点`underutilizedNodes`，节点有一个资源类型的request的利用率大于高水位阈值`highResourceThreshold`（包括不可调度节点）为高利用率节点`overutilizedNodes`，节点request的利用率介于`lowResourceThreshold`和`highResourceThreshold`之间为合理利用率节点`targetutilizedNodes`。

![descheduler-lowNodeUtilizatio-with-useDeviationThresholds-disable](https://midbai.com/post/descheduler-node-utilization-plugin/descheduler-lowNodeUtilizatio-useDeviationThresholds-disable.webp)

当`useDeviationThresholds`启用的时候，先计算所有节点的request平均使用率`averageResourceUsagePercent`，低水位阈值`lowResourceThreshold`为`thresholds - averageResourceUsagePercent`，高水位阈值`highResourceThreshold`为`targetThresholds + averageResourceUsagePercent`。节点的分类规则跟不启用`useDeviationThresholds`一样。

![descheduler-lowNodeUtilizatio-with-useDeviationThresholds-enable](https://midbai.com/post/descheduler-node-utilization-plugin/descheduler-lowNodeUtilizatio-useDeviationThresholds-enable.webp)

#### 1.为什么需要 useDeviationThresholds？——动态化

参考 issue：https://github.com/kubernetes-sigs/descheduler/pull/473。

主要是为了实现 threholds 和 targetThreholds 的动态配置。考虑到 LowNodeUtilization 策略是为了将高利用率的节点上的pod移动到低利用率的节点上，因此要防止当集群中整体的负载较高时，可能会出现的频繁的 pod 驱逐。

因此，为了避免这一点，需要拉高成为高负载 node 的阈值（因此需要 + averageResourceUsagePercent），使其更难成为高负载 node。

同时也要拉高成为低负载 node 的阈值（因此需要 - averageResourceUsagePercent），使其满足一旦成为低负载 node，就拥有更多的空间承接高负载 node 的负载。

#### 2.LowNodeUtilization 迁移效果分析

重调度功能完善之后，还需要一套效果评估机制，主要包括如下三个指标：

1. 第一个是高利用率节点的发现率，指的是二次调度能发现的高利用率节点数量，配置的报警敏感度低以及 Pod 迁移窗口限制等原因导致发现率是偏低的。
2. 第二个是高利用率节点 Pod 驱逐率，驱逐率指的是发现高利用率节点之后，能够在节点上挑选出合适的 Pod 并进行驱逐。有些高利用率节点因为服务相关约束、实例挑选算法不合理或者全局策略约束等原因无法驱逐节点上面的 Pod。
3. 第三个是高利用率节点 Pod 驱逐有效率。如果驱逐了高利用率节点上面的Pod节点利用率没有降低到一定阈值，那也是不符合预期的。主要有三个原因会导致驱逐有效率低，第一个是因为各种约束导致驱逐的 Pod 不是最优的，节点利用率下降不明显，第二个就是 Pod 被驱逐后，节点上很快会有新 Pod 被调度上来了，第三个是节点上部分 Pod 利用率变高了。

最后就是对全集群高利用率节点数量的分析，针对全集群高利用率节点的数据，也会按资源池与集群维度进行拆分建立以上三个指标数据，理论上高利用率节点发现率、驱逐率和有效率三个指标如果都比较符合预期全集群高利用率节点的比例是会控制在一定范围内的。

![img](https://cdn.tianfeiyu.com/kubernetes%2Fdescheduler-3.png)

### 3.扩展 LowNodeUtilization 和 HighNodeUtilization 策略

Descheduler 默认以 Job 或者 CronJob 的形式触发，这种周期性的巡检机制有一定的滞后性，真实场景下，如果节点利用率过高则要尽快进行处理。针对节点高利用率场景，为了提高时效性，策略在扩展时直接对接了内部的监控系统，**通过告警回调触发节点上面 Pod 的迁移**。

> 针对部分无法在高峰期进行 Pod 迁移的场景，可以设计节点利用率预测算法，在高峰期前提前预测高利用率节点执行 Pod 迁移操作。

## 3.Descheduler 的使用方法

 在 Kubernetes 集群内部通过 Job 或者 CronJob 的形式来运行 `Deschduler`，这样可以多次运行而无需用户手动干预，此外 `Descheduler` 的 Pod 在 `kube-system` 命名空间下面以 `critical pod` 的形式运行，可以避免被自身或者 `kubelet` 驱逐了。

> 什么是 critical pod？
>
> 在 Kubernetes 中，Critical Pod 是一种特殊类型的 Pod，它们被认为是对集群运行至关重要的 Pod。Critical Pod 具有高优先级，并且在节点上的其他工作负载之前运行。它们通常用于支持 Kubernetes 的关键功能或基础设施，例如网络插件、存储插件、DNS 服务等。
>
> 关键的 Pod 由 Kubernetes 系统组件自动创建和管理，它们在集群中的生命周期中扮演着重要的角色。这些 Pod 的故障可能会导致集群中的服务中断或不稳定，因此它们的可靠性和高可用性至关重要。

### 1.Descheduler 配置文件指南

> 以下内容来自于 Descheduler 官网（https://github.com/kubernetes-sigs/descheduler/blob/v0.28.1/README.md）。

Descheduler 的配置文件示例如下所示：

```yaml
apiVersion: "descheduler/v1alpha2"
kind: "DeschedulerPolicy"
nodeSelector: "node=node1" # you don't need to set this, if not set all will be processed
maxNoOfPodsToEvictPerNode: 5000 # you don't need to set this, unlimited if not set
maxNoOfPodsToEvictPerNamespace: 5000 # you don't need to set this, unlimited if not set
profiles:
  - name: ProfileName
    pluginConfig:
    - name: "DefaultEvictor"
      args:
        evictSystemCriticalPods: true
        evictFailedBarePods: true
        evictLocalStoragePods: true
        nodeFit: true
    plugins:
      # DefaultEvictor is enabled for both `filter` and `preEvictionFilter`
      # filter:
      #   enabled:
      #     - "DefaultEvictor"
      # preEvictionFilter:
      #   enabled:
      #     - "DefaultEvictor"
      deschedule:
        enabled:
          - ...
      balance:
        enabled:
          - ...
      [...]
```

nodeSelector、maxNoOfPodsToEvictPerNode、maxNoOfPodsToEvictPerNamespace 是三个顶级配置。

之后是 profiles，它是一个数组，里面的关键配置是 plugins 和 pluginConfig，二者中配置关系是一一对应的，plugins 里面有一个 plugin，pluginConfig 中就会有对应的 config，config 中的 args 用来配置相应 plugin 的参数。

> 从源码来看，这里一共有四个扩展点：
>
> ![四个扩展点](四个扩展点.png)
>
> 而插件则有六种：
>
> ![六种插件](六种插件.png)
>
> Plugins 是可以写在配置文件中的。
>
> 1. preSort。
> 2. Sort。
> 3. filter（默认启用了 DefaultEvictor 插件）。
> 4. preEvictionFilter（默认启用了 DefaultEvictor 插件）。
> 5. deschedule：这些插件逐一处理 Pod，并按顺序驱逐它们。
> 6. balance：这些插件处理所有 pod 或 pod 组，并根据组的预期传播方式确定要驱逐哪些 pod。。
>
> 编写新的插件时需要实现相应的扩展点（至少一个）。

#### 1.通过源码确定两个 nodeSelector 的作用是什么？

从源码来看，`default evictor plugin` 中的 `node selector` 属性是用于 `NodeFit` 属性的过滤的，也就是说，NodeFit 过滤策略中的预选 Node 是满足 `DefaultEvictor.NodeSelector` 属性的筛选条件的。



### 2.Pod 过滤

使用**命名空间**进行过滤。

* 部分策略接受 `namespace` 参数，该参数允许相应的插件只作用于或者不作用于相应的命名空间。
* 部分策略接受 `evictableNamespaces` 参数，该参数允许相应的插件不作用于相应的命名空间。

使用**优先级**进行过滤。

> Pod 优先级相关的概念：https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/pod-priority-preemption/。
>
> 优先级表示一个 Pod 相对于其他 Pod 的重要性。 如果一个 Pod 无法被调度，调度程序会尝试抢占（驱逐）较低优先级的 Pod， 以使悬决 Pod 可以被调度。

可以通过默认驱逐过滤器配置优先级阈值，并且只有低于阈值的 Pod 才能被驱逐。您可以通过设置 `priorityThreshold.name` （将阈值设置为给定优先级类别的值）或 `priorityThreshold.value` （直接设置阈值）参数来指定此阈值。

使用**标签**进行过滤。

> k8s 中的 label 选择器主要有两种 matchLabels 和 matchExpressions。

部分插件允许使用标签进行过滤，选择出特定的 Pod 执行相应的策略。

使用**节点预匹配**进行过滤。

NodeFit 可以通过默认 Evictor Filter 进行配置。如果设置为 `true` ，调度程序将在驱逐它们之前考虑满足驱逐标准的 Pod 是否适合其他节点。如果 Pod 无法重新调度到另一个节点，则它不会被驱逐。目前，将 `nodeFit` 设置为 `true` 时考虑以下标准：

- Pod 上的 `nodeSelector`。
- Pod 上的任何 `tolerations` 以及其他节点上的任何 `taints`。
- Pod 上的 `nodeAffinity`。
- Pod 生成的资源 `requests` 以及其他节点上可用的资源。
- 是否有任何其他节点被标记为 `unschedulable`。

> 节点预匹配是基于当前 Pod 的 Spec 的，而不是其所有者（例如 ReplicationController）的 Spec，因此，如果其所有者最近被修改过，那么 Pod 可能依旧使用过去的 Spec。为了避免这一点，应该尽量使用 Deployment 而不是 ReplicationControllers。

## 4.Descheduler 的 Pod 驱逐机制





## 5.Descheduler高可用性





## 6.Descheduler 执行流程

### 1.为什么会多出 preEvictionFilter 阶段？

























## 7.Descheduler 的不足

**基于 request 计算节点负载并不能反映真实情况**

Descheduler 是通过 Node 上 Pod 的 Request 值来计算节点负载情况的。

这种方式是不太适合真实场景的，如果能直接拿 metrics-server 或者 prometheus 中的数据，会更有意义，因为很多情况下 Request、Limit 设置都不准确。有时，为了节约成本提高部署密度，Request 甚至会设置为 50m，甚至 10m。

**驱逐 Pod 导致应用不稳定**

Descheduler 通过策略计算出一系列符合要求的 Pod 进行驱逐。好的方面是，descheduler 不会驱逐没有副本控制器的 Pod，不会驱逐带本地存储的 Pod 等，保障在驱逐时，不会导致应用故障。但是 Descheduler 会直接删掉所有被选出的 Pod，然后再重新启动，而不是滚动更新。

这可能会造成，在一个短暂的时间内，在集群上可能没有 Pod 就绪，或者因为故障新的 Pod 起不来，服务就会报错。

**依赖于 Kubernetes 的调度策略**

Descheduler 并没有实现调度器，而是依赖于 Kubernetes 的调度器。这也意味着，descheduler 能做的事情只是驱逐 Pod，让 Pod 重新走一遍调度流程。如果节点数量很少，descheduler 可能会频繁的驱逐 Pod。

## 8.Descheduler 约束策略

由于生产环境中场景复杂，Pod 迁移对业务来说也是一个有损的操作，在迁移过程中必须要做好必要的防范措施，需要配置一些约束策略来保障业务的稳定性。

尽管 k8s 可以通过配置 PDB 来避免对象的副本被同时驱逐，不过我们认为 PDB 不够精细化，在跨集群场景中也无法更好的运用，此处会通过一个全局的约束限制模块让服务的 Pod 在重调度过程中对服务影响尽可能的小以及尽可能的均衡负载有问题的节点，保证业务不中断或业务 SLA 不降级。

> k8s PDB 是什么？
>
> 在 Kubernetes 中，PDB 是 PodDisruptionBudget（Pod 故障容忍度预算）的缩写。PodDisruptionBudget 是一个 Kubernetes 的资源对象，用于控制在进行节点维护或调度故障时，允许同时终止的 Pod 的数量。
>
> PDB 允许您定义了阈值，指定了在某些情况下可以同时终止的 Pod 的最大数量。这些情况通常是因为节点维护、故障、调度或其他原因而导致需要关闭节点上的 Pod。通过使用 PDB，**您可以确保集群中的应用程序在这些情况下仍能够保持正常运行，而不至于因为过多的 Pod 被终止而导致服务不可用或性能下降**。

当前在 Pod 迁移过程中有多种策略来进行约束。

1. 在**宏观策略**上，主要是针对全局的约束策略，在不同的集群与资源池中，Pod 迁移时的速率以及一个周期内迁移 Pod 总数量会有限制，Pod 迁移时间窗口、迁移时是否跨集群等策略也有一定的限制，每个集群与资源池也会配置黑白名单。
2. 在**微观策略**上，主要是针对节点和服务的约束策略，节点与服务 Pod 迁移速率与一个周期内迁移 Pod 总数量也有限制，在迁移时挑选服务下 Pod 也会针对 Pod 状态以及服务等级做一些限制。

# 附录

## 1.什么是 Node Allocatable 和 Node Capacity

字段 `.status.allocatable` 描述节点上可以用于 Pod 的资源总量（例如：15 个虚拟 CPU、7538 MiB 内存）。关于 Kubernetes 中节点可分配资源的信息， 可参阅[为系统守护进程预留计算资源](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/reserve-compute-resources/)。

> Kubernetes 节点上的 'Allocatable' 被定义为 Pod 可用计算资源量。 调度器不会超额申请 'Allocatable'。 目前支持 'CPU'、'memory' 和 'ephemeral-storage' 这几个参数。

字段 `.status.capacity` 描述了节点上总的资源总量，一般大于 allocatable。

> Kubernetes 的节点可以按照 `Capacity` 调度。默认情况下 pod 能够使用节点全部可用容量。 这是个问题，因为节点自己通常运行了不少驱动 OS 和 Kubernetes 的系统守护进程。 除非为这些系统守护进程留出资源，否则它们将与 Pod 争夺资源并导致节点资源短缺问题。

allocatable 和 capacity 之间的关系如下：

![allocatable 和 capacity 之间的关系](allocatable 和 capacity 之间的关系.png)

这两个属性可以参照如下指令获取：

```bash
$ kubectl describe nodes e2e-test-node-pool-4lw4

Name:            e2e-test-node-pool-4lw4
[ ... 这里忽略了若干行以便阅读 ...]
Capacity:
 cpu:                               2
 memory:                            7679792Ki
 pods:                              110
Allocatable:
 cpu:                               1800m
 memory:                            7474992Ki
 pods:                              110
[ ... 这里忽略了若干行以便阅读 ...]
Non-terminated Pods:        (5 in total)
  Namespace    Name                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------    ----                                  ------------  ----------  ---------------  -------------
  kube-system  fluentd-gcp-v1.38-28bv1               100m (5%)     0 (0%)      200Mi (2%)       200Mi (2%)
  kube-system  kube-dns-3297075139-61lj3             260m (13%)    0 (0%)      100Mi (1%)       170Mi (2%)
  kube-system  kube-proxy-e2e-test-...               100m (5%)     0 (0%)      0 (0%)           0 (0%)
  kube-system  monitoring-influxdb-grafana-v4-z1m12  200m (10%)    200m (10%)  600Mi (8%)       600Mi (8%)
  kube-system  node-problem-detector-v0.1-fj7m3      20m (1%)      200m (10%)  20Mi (0%)        100Mi (1%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests    CPU Limits    Memory Requests    Memory Limits
  ------------    ----------    ---------------    -------------
  680m (34%)      400m (20%)    920Mi (11%)        1070Mi (13%)
```















