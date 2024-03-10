---
layout: mypost
title: Kubernetes 调度策略
categories: [Kubernetes]
---

## 1.k8s 调度策略

Kubernetes 提供了四大调度方式：

> 以下属性中，除了 Taints 是 Node 的属性外，Toleration 则是 Pod 的属性。

1. 自动调度，运行在哪个节点上完全由Scheduler经过一系列的算法计算得出。
2. 定向调度：NodeName、NodeSelector。
3. 亲和性调度：NodeAffinity、PodAffinity、PodAntiAffinity。
4. 污点（容忍）调度：Taints、Toleration。

# 2.Pod 拓扑分布约束

要弄明白 Pod 拓扑分布约束，首先需要定义拓扑键。拓扑键的定义取决于节点标签，它们可以标识每个节点所在的拓扑域。

假设我们一个 4 节点的集群带有如下标签：

```bash
NAME    STATUS   ROLES    AGE     VERSION   LABELS
node1   Ready    <none>   4m26s   v1.16.0   node=node1,zone=zoneA
node2   Ready    <none>   3m58s   v1.16.0   node=node2,zone=zoneA
node3   Ready    <none>   3m17s   v1.16.0   node=node3,zone=zoneB
node4   Ready    <none>   2m43s   v1.16.0   node=node4,zone=zoneB
```

那么，从逻辑上看集群如下所示：

![集群的逻辑分布](集群的逻辑分布.png)

# 3.Pod 亲和性与反亲和性中的 topologyKey 是什么

下面是一个 Pod 亲和性实例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: registry.k8s.io/pause:2.0
```

亲和性规则规定，只有节点属于特定的区域且该区域中的其它 Pod 已经打上 `security=S1` 标签时，调度器才可以将示例 Pod 调度到此节点上。

反亲和性规则规定，如果节点属于特定的区域且该区域中的其他 Pod 已打上 `security=S2` 标签，则调度器应尝试避免将 Pod 调度到此节点上。

> 这里的 topologyKey 和 labelSelector 是**且**的关系。

# 4.Node 的污点

在 Kubernetes 中，Node 污点的结构是 `k=v:t`，其中：

- `k` 表示键（Key），它是一个字符串，用于标识污点的名称或属性。
- `v` 表示值（Value），它是一个字符串，用于指定污点的值。这通常用于对污点进行更具体的描述。
- `t` 表示效果（Effect），它定义了污点的作用效果。常见的效果包括：
  - `NoSchedule`：阻止新的 Pod 被调度到该节点上。
  - `PreferNoSchedule`：不鼓励但允许新的 Pod 被调度到该节点上。
  - `NoExecute`：从节点上移除现有的 Pod。

在这种结构中，`k=v` 对的作用是将污点更具体地标识和描述。

当节点被标记上污点时，它会影响该节点上所有尝试被调度的 Pod，污点的作用是节点级别的，而不是针对单个 Pod 的。

# 5.什么是 Kubernetes Operator？

Operator = CRD(CustomResourceDeftination) + AdmissionWebhook + Controller









































