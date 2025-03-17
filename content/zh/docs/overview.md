---
weight: 100
title: "总览"
description: ""
icon: "travel_explore"
date: "2025-03-16T20:13:34+08:00"
lastmod: "2025-03-16T20:13:34+08:00"
draft: false
toc: true
---

## OGraph 是什么

OGraph 是一个专为 gopher 设计的有向无环图（DAG）调度框架，利用了 Go 协程的调度能力。在保持简洁的同时，它提供了高性能和强大的功能。

你可以用它来构建和执行工作流，就和 [taskflow](https://github.com/taskflow/taskflow) 和 [CGraph](https://github.com/ChunelFeng/CGraph) 所做的事情一样。如果你正在为实现并行计算或者多任务调度寻找工作流引擎，并且基于Go技术栈，那么 OGraph 或许就是答案。

和一些工作流引擎不同，OGraph 不是一个可以直接部署运行的独立服务。具体来说就是你不能通过部署 OGraph 获取一个可以提交、执行和查看任务的 Web 界面。它在更低一层，可以作为这样一个服务的核心功能实现。当然，更主要的场景还是直接在代码中编排任务，来完成特定业务的一些复杂业务流程，比如视频流处理等。

## 为什么用 OGraph

首先我要说明的是，OGraph 不是其他语言类似项目的移植。而是按 Go 风格，最大化利用 Go 优势的一个项目。

Go 协程背后本身就具有一个复杂精巧的调度系统，建立在 go 协程调度系统之上，OGraph 可以轻松获取和其他基于线程池的项目相当的性能，并且更加轻量灵活。

简单归纳 OGraph 的优势有以下几点：

{{< table "table-striped" >}}
|              |                                                                                           |
| ------------ | ----------------------------------------------------------------------------------------- |
| `快速`       | O性能超越其他 Go 框架 10 倍。对于节点少于 64 的一般任务流，调度成本低于 10 微秒。         |
| `友好`       | 包含强大且多用途的节点和包装器，开箱即用。它还附带了丰富的文档和示例来帮助你快速上手。    |
| `丰富特性`   | 除了基本功能外，还支持图导入和导出、暂停、事务、事件监听、优先级调度以及性能分析等功能... |
| `高度可定制` | 关键组件被抽象为接口，在特殊场景下你可以用自己实现来替换框架提供的默认实现。              |
| `易于伸缩`   | 得益于 Go 的调度机制，你可以在不同的硬件资源条件下充分利用资源而无需进行调优。            |
| `Stability`  | 最小化第三方依赖，精心设计新特性。稳定性是 ograph 的重要目标之一。                        |
{{< /table >}}

## 架构

在 OGraph 中一切基于 Node，Node 是可运行的业务逻辑的抽象。

其中 Pipeline 是一个特殊的 Node，它本身由其他 Node 组成，并且记录了各自的依赖关系。运行 Pipeline 就是按依赖关系依次序执行这些 Node。自然而然的，Pipeline 可以包含子 Pipeline，因为 子 Pipeline 也是 Node，只要是 Node 就可以被 Pipeline 管理执行。

除了 Pipeline，还有 Wrapper 和 Cluster 可以包含其它 Node ，Wrapper 用于增强被包含节点的功能（比如增加日志打印），Cluster 用于补充更灵活的调度方式。

以上提到的，只有 Pipeline 是一个具体实现，Node、Wrapper、Cluster 都是接口，你可以自定义实现。

一个 Pipeline 可以用下图表示：

![overview.png](/images/overview.png)

黄色的框代表 Pipeline，可以看到 Pipeline 内包含许多 Node，包括用户自定义的 Node，一个为 Node 增加循环执行功能的 Wrapper，一个选择特定 Node 执行的 Cluster，以及一个子 Pipeline。

就像前面说的，Pipeline 可以包含其他任何 Node，Wrapper、Cluster 也是如此。所以你还可以看到 Wrapper 包含 Pipeline，Cluster 包含 Wrapper 等等情况。这将带给你最灵活的功能组合。

Node 之间用箭头连接，表示依赖关系。没有任何依赖的 Node 首先执行，然后执行它的下游Node。

