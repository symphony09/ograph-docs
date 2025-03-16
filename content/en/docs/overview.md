---
weight: 100
title: "Overview"
description: ""
icon: "travel_explore"
date: "2025-03-16T22:17:18+08:00"
lastmod: "2025-03-16T22:17:18+08:00"
draft: false
toc: true
---

## What is OGraph

A **modern**, **DAG-based** multi-task scheduling framework for **parallel computing** or **executing workflows**. It offers the **simplicity and flexibility** of Go, combined with the **speed** of C++.

You can use it to build and execute workflows, just like what [taskflow](https://github.com/taskflow/taskflow) and [CGraph](https://github.com/ChunelFeng/CGraph) do. If you are looking for a workflow engine to implement parallel computing or multitasking scheduling and are working within the Go technology stack, OGraph might be the solution.

Unlike some workflow engines, OGraph is not a standalone service that can be deployed and run directly. Specifically, you cannot obtain a web interface for submitting, executing, and viewing tasks by deploying OGraph. It operates at a lower level and can be used as the core functionality to implement such services. Of course, the primary use case is to orchestrate tasks directly within code to accomplish complex business processes for specific businesses, such as video stream processing.

## Why OGraph

First, I need to clarify that OGraph is not a port of similar projects in other languages. Instead, it is a project designed with the Go style and maximizes the advantages of the Go language.

The Go language itself has a sophisticated and intricate scheduling system behind its goroutines. Built on top of the Go coroutine scheduling system, OGraph can easily achieve performance comparable to other projects based on thread pools but with greater lightweightness and flexibility.

In summary, the advantages of OGraph are as follows:

{{< table "table-striped" >}}
|                       |                                                                                                                                                                                    |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Fast`                | Outperforms other Go frameworks by 10 times.For general task flows with less than 64 nodes, the scheduling cost is under 10 microseconds.                                          |
| `Friendly`            | Includes powerful and versatile nodes and wrappers, ready to use upon opening the box. It also comes with rich documentation and examples to help you get started.                 |
| `Rich features`       | In addition to basic features, it also supports graph import and export, pause, transactions, event listening, priority scheduling, and performance analysis...                    |
| `Highly customizable` | Key components are abstracted as interfaces, which allows you to replace the default implementations provided by the framework with your own implementations in special scenarios. |
| `Easy to scale`       | Thanks to Go's scheduling mechanism, you can fully utilize resources under different hardware resources without the need for tuning.                                               |
| `Stability`           | Minimal third-party dependencies, carefully designed new features. Stability is one of the important goals of ograph.                                                              |
{{< /table >}}

## Architecture

In OGraph, everything is based on Nodes. A Node is an abstraction of executable business logic.

A Pipeline is a special type of Node that itself consists of other Nodes and records their dependency relationships. Running a Pipeline involves executing these Nodes in order according to their dependencies. Naturally, Pipelines can contain sub-Pipelines because sub-Pipelines are also Nodes, and any Node can be managed and executed by a Pipeline.

In addition to Pipelines, there are Wrappers and Clusters that can also include other Nodes. A Wrapper is used to enhance the functionality of included nodes (such as adding log printing), while a Cluster provides more flexible scheduling methods.

Of the above mentioned components, only Pipelines have specific implementations, whereas Nodes, Wrappers, and Clusters are interfaces that you can customize and implement yourself.

A Pipeline can be represented by the following diagram:

![overview.png](/images/overview.png)

A yellow box represents a Pipeline, which can be seen to contain many Nodes, including user-defined Nodes, a Wrapper that adds loop execution functionality to a Node, a Cluster that selects specific Nodes for execution, and a sub-Pipeline.

As mentioned earlier, Pipelines can include any other type of Node (Nodes, Wrappers, Clusters), and the same applies to Wrappers and Clusters. Therefore, you may also see a Wrapper containing a Pipeline or a Cluster containing a Wrapper, providing you with the most flexible combination of features.

Nodes are connected by arrows to indicate dependency relationships. Nodes with no dependencies will be executed first, followed by their downstream Nodes.