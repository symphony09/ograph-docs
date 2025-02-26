---
weight: 201
title: "Cluster"
description: "A guide to the cluster feature."
icon: "article"
date: "2025-02-23T15:31:57+08:00"
lastmod: "2025-02-23T15:31:57+08:00"
draft: false
toc: true
---

In OGraph, cluster is a special type of node that contains several child nodes. The scheduling of these child nodes is controlled by the cluster, rather than by the pipeline.

The purpose of the cluster is to provide a more flexible scheduling method beyond dependency-driven, such as selecting one node among multiple nodes to execute.

## Build cluster

All clusters are constructed by the [Factory](./factory.md). Unlike constructing nodes and wrappers, clusters need to be constructed not only themselves but also their child nodes.

Therefore, you need to pass the child nodes in the `UseFactory` method in addition. But the child nodes do not need to use the factory pattern.

**Example**

```go
	n1 := ograph.NewElement("N1").UseFn(func() error {
		fmt.Println("N1 running")
		return nil
	})
	n2 := ograph.NewElement("N2").UseFn(func() error {
		fmt.Println("N2 running")
		return nil
	})

	c := ograph.NewElement("C1").UseFactory(ogimpl.Choose, n1, n2).Params("ChooseExpr", "index")
```

In the example code, n1 and n2 are regular nodes, and c1 is a cluster node. In the code declaring c1, the `UseFactory` method is extra-passed with n1 and n2. Therefore, after the factory builds c1, it will also inject n1 and n2 into c1.

## Use cluster

Using a cluster is the same as using a regular node.

## Customize cluster

Here, we will use a custom reversed execution cluster as an example.

**Define a factory**

```go
type CustomCluster struct {
	ograph.BaseCluster
}

func (cluster *CustomCluster) Run(ctx context.Context, state ogcore.State) error {
	for i := len(cluster.Group) - 1; i >= 0; i-- {
		if err := cluster.Group[i].Run(ctx, state); err != nil {
			return err
		}
	}
	return nil
}

func NewCustomCluster() ogcore.Node {
	return &CustomCluster{}
}
```

Define a cluster like defining a node, but it must implement the `Join(nodes []ogcore.Node)` method. The example inherits from `ograph.BaseCluster`, so you don't need to implement it yourself.

After inheriting from BaseCluster, you can obtain all child nodes of the cluster through the `Group` field (in the declared order).

**Use in the pipeline**

```go
func TestCluster_Customize(t *testing.T) {
	pipeline := ograph.NewPipeline()

	pipeline.RegisterFactory("ReverseOrder", NewCustomCluster)

	first := ograph.NewElement("first").UseFn(func() error {
		fmt.Println("first")
		return nil
	})

	second := ograph.NewElement("second").UseFn(func() error {
		fmt.Println("second")
		return nil
	})

	myCluster := ograph.NewElement("MyCluster").
		UseFactory("ReverseOrder", first, second)

	pipeline.Register(myCluster)

	if err := pipeline.Run(context.TODO(), nil); err != nil {
		t.Error(err)
	}
}
```

1. Register [factory](./factory.md) of cluster.
2. Declare cluster element and register it to pipeline.

**Output**
```
second
first
```

