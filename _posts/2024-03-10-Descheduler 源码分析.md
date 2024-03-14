---
layout: mypost
title: Descheduler 源码分析
categories: [Descheduler, Kubernetes]
---

## 1.从 cmd 下面的 descheduler.go 作为入口文件。

## 2.通过一个 cobra 工具创建了一个 cmd。

cmd 中的 RunE 是主要的执行流程，其中定义了安全服务器（用于认证和鉴权）、可观测性服务器（基于 opentelemetry）。

## 3.之后进入 Descheduler 的 Run 函数中（主流程）

1. 通过和 go-client 与 Kubernetes 集群取得通信。
2. 加载 Policy 的配置（读取 .yaml 配置文件）。
3. 执行重调度策略。

## 4.执行重调度策略（RunDeschedulerStrategies）

### 1.什么是 dry run？

在 k8s 中，dry run 是一种测试资源创建、更新或删除操作的机制。

### 2.eventClient 和 client 的区别

综上所述，Client 是一个通用的 Kubernetes API 客户端，用于管理各种类型的资源对象，而 EventClient 则是专门用于处理 Kubernetes 事件的客户端，用于访问和管理事件流中的事件。

> - Kubernetes 事件是指与 Kubernetes 资源对象相关的状态变化或其他重要事件，如 Pod 创建、删除、调度等。事件通常记录在 Kubernetes 的事件流中，并可以通过 EventClient 进行访问。
> - EventClient 提供了用于访问和管理事件的方法，用户可以使用它来查询事件流中的事件，监视事件的发生，并进行相应的处理。

### 3.当父上下文（`ctx`）被取消时，派生的所有子上下文也会被取消

### 4.顶层的 NodeSelector 的作用

依据源码可以看出，其作用是通过 `LabelSelector` 字段筛选符合特定标签条件的节点。

## 5.在 wait.NonSlidingUntil 中执行 Descheduler 重调度

## 6.在 Descheduler 中的 runDeschedulerLoop 执行具体的策略

在该方法中执行了 `evictions.NewPodEvictor`，创建了一个新的驱逐器（这个驱逐器会最终在执行相应的插件逻辑时用到）。

之后执行 runProfiles 方法。

## 7.在 Descheduler 的 runProfiles 中运行调度策略

### 1.go range 一个 struct 会发生什么?

在 Go 中，使用 `range` 关键字遍历一个结构体（struct）会得到结构体中的字段值。当使用 `range` 遍历结构体时，实际上是遍历结构体中的字段（**KV 结构**），而不是遍历结构体本身。

### 2.struct 的同类声明

如下代码所示：

```go
type profileRunner struct {
	name                      string
	descheduleEPs, balanceEPs eprunner
}
```

profileRunner 中包含了两个 eprunner 类型的字段，descheduleEPs, balanceEPs。

> `type eprunner func(ctx context.Context, nodes []*v1.Node) *frameworktypes.Status`。

### 3.newProfile 做了什么

> newProfile 中定义了 handle，它持有驱逐器，会被传到 Plugin 的 Builder 中。

newProfile 主要是将 yaml 中的 Profile 转化为了内存对象，包括注入驱逐器，注入相应的插件。 

同时 newProfile 的返回对象存在 RunDeschedulePlugins 方法和 RunBalancePlugins，二者是执行相应插件的。

最终 runProfiles 中会调用这两个方法并执行相应的插件代码。

## 8.在 Descheduler 的 runProfiles 中运行调度策略

具体的运行函数是 Profile 中的 RunDeschedulePlugins 和 RunBalancePlugins。

## 9.驱逐程序

驱逐代码主要是在各个插件中提供。

驱逐原理是 k8s 的驱逐原理，可以去了解一下。






