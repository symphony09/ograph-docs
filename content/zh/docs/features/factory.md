---
weight: 201
title: "工厂模式"
description: "关于工厂模式特性的指南"
icon: "article"
date: "2025-03-09T21:01:19+08:00"
lastmod: "2025-03-09T21:01:19+08:00"
draft: false
toc: true
---

在ograph中，你可以直接将要执行的节点传递给`pipeline`，也可以告诉`pipeline`如何构建你想要执行的节点。

前者可以视为单例模式，而后者是本文的重点，即介绍工厂模式。

使用工厂模式具有以下优点：

1. **懒加载**。这允许你在需要时按需创建节点，无需提前预建它们。`pipeline`可以在实际需要时才创建节点，这样更高效且节省资源。
2. **并发安全**。ograph支持`pipeline`的并行执行，但如果同一节点在并发情况下被执行，则可能会出现问题。为了解决这个问题，在并发执行时使用工厂模式构建独立节点。
3. **支持导出图**。可以在任何有相应工厂的地方创建节点，因此可以将一个`pipeline`中的节点导出并在另一个`pipeline`中导入以供执行。


{{< alert context="info" text="包装器和簇只能通过工厂创建，不能用其他方法构建。" />}}

## 示例

```go
type ProductionX struct {
	Date time.Time
}

func (p ProductionX) Run(ctx context.Context, state ogcore.State) error {
	fmt.Printf("produced at %s\n", p.Date)
	return nil
}

func NewProductionX() ogcore.Node {
	r := rand.New(rand.NewSource(time.Now().Unix()))
	time.Sleep(time.Duration(r.Intn(10)) * time.Millisecond)
	return &ProductionX{Date: time.Now()}
}
```

首先，定义一个节点类型。函数`NewProductionX`用于创建节点实例，这就是我们需要的工厂函数。

接下来，我们需要注册并使用这个工厂。

```go
func TestFactory(t *testing.T) {
	pipeline := ograph.NewPipeline()

    // 注册到 pipeline 中
	pipeline.RegisterFactory("X", NewProductionX)

    // 通过引用工厂名称来使用已注册的工厂。
	x1 := ograph.NewElement("x1").UseFactory("X")
	x2 := ograph.NewElement("x2").UseFactory("X")

	pipeline.Register(x1).Register(x2)

	if err := pipeline.Run(context.TODO(), nil); err != nil {
		t.Error(err)
	}
}
```

**输出**

```
produced at 2025-02-25 23:15:16.2117526 +0800 CST m=+0.006252301
produced at 2025-02-25 23:15:16.2102107 +0800 CST m=+0.004710401
```

## 设置参数

大多数情况下，我们需要灵活地使用不同参数构建节点，而不是每次都构建相同的节点。

在上面的例子中，我们能否通过参数控制ProductionX的Date属性？答案是肯定的，并且非常简单。

只需要修改一行代码：

```go
x1 := ograph.NewElement("x1").UseFactory("X").Params("Date", time.Now().Add(time.Hour))
```

`Params`方法包含两个参数：一个是属性名称，另一个是属性值。

{{< alert context="info" text="为了使属性更改生效，节点必须是指针类型。" />}}

## 自定义初始化

有时候，仅仅设置参数（属性）是不够的。可能还需要执行额外的初始化操作，例如检查属性是否有效。这在ograph中也是支持的。

在上述示例中，初始化检查将如下所示：

```go
func (p *ProductionX) Init(params map[string]any) error {
	productDate, _ := params["Date"].(time.Time)
	if productDate.After(time.Now()) {
		return fmt.Errorf("invalid date")
	} else {
		p.Date = productDate
	}
	return nil
}
```

你只需要为节点实现一个 `Init` 方法，参数 `params` 来自通过 `Element.Params` 设置的键值对。

## 注册全局工厂

你可能已经注意到，在使用 ograph 内置的包装器和簇时，它们不需要在`pipeline`中注册。这是因为导入 `ogimpl` 包时，这些工厂会自动注册到全局工厂中。

所有`pipeline`都可以访问全局工厂，因此无需再在`pipeline`中重新注册工厂。

注册全局工厂的代码如下所示：

```go
import (
	"github.com/symphony09/ograph/global"
)

// ...
    global.Factories.Add("FactoryName", YourFactory)
// ...
```

{{< alert context="warning" text="使用相同名称注册工厂将会覆盖旧的工厂。" />}}