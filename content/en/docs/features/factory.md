---
weight: 201
title: "Factory"
description: "A guide to the factory feature."
icon: "article"
date: "2025-02-23T13:57:09+08:00"
lastmod: "2025-02-23T13:57:09+08:00"
draft: false
toc: true
---

In ograph, you can pass the nodes you want to execute directly to the pipeline, or you can tell the pipeline how to build the nodes you want to execute.

The former can be considered a singleton pattern, while the latter is the focus of this document, which is to introduce the factory pattern.

Using the factory pattern has the following advantages:

1. **Lazy loading**. This allows you to create nodes on-demand, without having to pre-build them in advance. The pipeline can create nodes only when they are needed, which can be more efficient and cost-effective.
2. **Concurrent safety**.  ograph supports concurrent execution of the pipeline, but if the same node is executed concurrently, it can cause problems. To solve this issue, independent nodes are built using the factory pattern when executing concurrently.
3. **Support for exporting graphs**. Nodes can be created anywhere where there is a corresponding factory, so they can be exported from one pipeline and imported into another for execution.

{{< alert context="info" text="wrappers and clusters can only be created by the factory and cannot be built by other methods." />}}

## Example

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

First, define a node type. The function `NewProductionX` to create node instances is the factory function we need.

Next, we need to register and use this factory.

```go
func TestFactory(t *testing.T) {
	pipeline := ograph.NewPipeline()

    // Register it to pipeline
	pipeline.RegisterFactory("X", NewProductionX)

    // Use the registered factory by referring to the factory name.
	x1 := ograph.NewElement("x1").UseFactory("X")
	x2 := ograph.NewElement("x2").UseFactory("X")

	pipeline.Register(x1).Register(x2)

	if err := pipeline.Run(context.TODO(), nil); err != nil {
		t.Error(err)
	}
}
```

**Output**

```
produced at 2025-02-25 23:15:16.2117526 +0800 CST m=+0.006252301
produced at 2025-02-25 23:15:16.2102107 +0800 CST m=+0.004710401
```

## Set Parameters

Most of the time, we need to build nodes with different parameters flexibly, rather than building the same node every time.

In the example above, can we control the `Date` attribute of `ProductionX` with parameters? The answer is yes, and it's very simple.

Only one line of code needs to be modified:

```go
x1 := ograph.NewElement("x1").UseFactory("X").Params("Date", time.Now().Add(time.Hour))
```

The `Params` method contains two parameters: one for the attribute name, and one for the attribute value.

{{< alert context="info" text="In order for attribute changes to take effect, the node must be of pointer type." />}}

## Custom initialization

Sometimes, just setting the parameters (attributes) is not enough. Additional initialization operations may need to be performed. For example, checking whether the attributes are valid. This is also supported in ograph.

In the above example, the initialization check would look like this:

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

You only need to implement an `Init` method for the node, and the parameter `params` is set to a key-value pair through `Element.Params`.

## Register a global factory

You may have noticed that when using the built-in wrapper and cluster functions in ograph, they are not registered in the pipeline. This is because when importing the `ogimpl` package, these factories are automatically registered in the global factory.

All pipelines can access the global factory, so there is no need to register the factory again in the pipeline.

The code to register the global factory is as follows:

```go
import (
	"github.com/symphony09/ograph/global"
)

// ...
    global.Factories.Add("FactoryName", YourFactory)
// ...
```

{{< alert context="warning" text=" Registering a factory with the same name will override the old factory." />}}