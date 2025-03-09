---
weight: 201
title: "节点簇"
description: "关于节点簇特性的指南"
icon: "article"
date: "2025-03-09T20:50:18+08:00"
lastmod: "2025-03-09T20:50:18+08:00"
draft: false
toc: true
---

在OGraph中，`Cluster`是一种特殊类型的节点，包含多个子节点。这些子节点的调度由`Cluster`控制，而不是由`pipeline`控制。

`Cluster`的目的在于提供一种更加灵活的调度方法，而不仅仅是依赖驱动的调度方式，例如可以在多个节点中选择一个节点来执行。

## 构建簇

所有`Cluster`都是通过[工厂](./factory.md)构建的。与构建节点和包装器不同，`Cluster`不仅需要构建自身，还需要构建其子节点。

因此，在`UseFactory`方法中除了传递工厂本身外，还需要传递子节点。但是子节点可以不使用工厂模式。

**例子**

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

在示例代码中，n1和n2是常规节点，而c1是一个簇节点。在声明c1的代码中，`UseFactory`方法额外传递了n1和n2。因此，在工厂构建c1之后，还会将n1和n2注入到c1中。

## 使用簇节点

使用簇节点与使用常规节点相同。

## 自定义簇节点

这里我们将用一个自定义的逆序执行簇作为示例。

**定义簇的工厂**

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

定义一个簇节点类似于定义一个节点，但必须实现`Join(nodes []ogcore.Node)`方法。示例中的簇继承自`ograph.BaseCluster`，因此你不需要自己实现它。

继承BaseCluster后，你可以通过`Group`字段（按声明的顺序）获取簇的所有子节点。

**在 pipeline 中使用**

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

1. 注册[工厂](./factory.md)。
2. 声明集群元素并将其注册到`pipeline`中。

**输出**
```
second
first
```