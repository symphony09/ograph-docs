---
weight: 101
title: "Quick Start"
description: "A guide to getting up and running a pipeline."
icon: "rocket_launch"
date: "2025-02-22T21:19:07+08:00"
lastmod: "2025-02-22T21:19:07+08:00"
draft: false
toc: true
---

## Install

```shell
go get github.com/symphony09/ograph
```

## Demo

{{< prism lang="go" line-numbers="true" line="13-14" >}}
func TestDemo(t *testing.T) {
	pipeline := ograph.NewPipeline()

	n1 := ograph.NewElement("N1").UseFn(func() error {
		fmt.Println("N1 running")
		return nil
	})
	n2 := ograph.NewElement("N2").UseFn(func() error {
		fmt.Println("N2 running")
		return nil
	})

	pipeline.Register(n1).
		Register(n2, ograph.Rely(n1))

	if err := pipeline.Run(context.TODO(), nil); err != nil {
		t.Error(err)
	}
}
{{< /prism >}}

As shown in lines 13 to 14 of the code, two nodes are registered in the pipeline, and N2 is dependent on N1.

```mermaid
---
title: Demo pipeline
---

flowchart LR
	A[Start] --> N1
    N1((N1)) --> N2((N2))
	N2 --> B[End]

```

**Output**

```
N1 running
N2 running
```

## Loop

{{< tabs tabTotal="2">}}
{{% tab tabName="Diff Code" %}}

```go
// ...
n1 := ograph.NewElement("N1").UseFn(func() error {
		fmt.Println("N1 running")
		return nil
	}).Apply(ogimpl.LoopOp(3)) // Execute N1 3 times in a loop.
// ...
```

{{% /tab %}}

{{% tab tabName="Complete Demo" %}}

```go
func TestDemo2(t *testing.T) {
	pipeline := ograph.NewPipeline()

	n1 := ograph.NewElement("N1").UseFn(func() error {
		fmt.Println("N1 running")
		return nil
	}).Apply(ogimpl.LoopOp(3)) // Execute N1 3 times in a loop.

	n2 := ograph.NewElement("N2").UseFn(func() error {
		fmt.Println("N2 running")
		return nil
	})

	pipeline.Register(n1).
		Register(n2, ograph.Rely(n1))

	if err := pipeline.Run(context.TODO(), nil); err != nil {
		t.Error(err)
	}
}
```

{{% /tab %}}
{{< /tabs >}}

```mermaid
---
title: Demo pipeline 2
---

flowchart LR
	A[Start] --> N1
    N1((N1)) --> N2((N2))
	N1 -- 3 --> N1
	N2 --> B[End]
```

The loop is executed through the [Wrapper](features/wrapper.md), which implements functions such as conditional execution, timeout control, and error retry.

## Choose

{{< tabs tabTotal="2">}}
{{% tab tabName="Diff Code" %}}

```go
	// ...
	c := ograph.NewElement("C1").UseFactory(ogimpl.Choose, n1, n2).Params("ChooseExpr", "index")
	
	pipeline.Register(c)

	state := ograph.NewState()
	state.Set("index", 1) // choose node by index

	if err := pipeline.Run(context.TODO(), state); err != nil {
		t.Error(err)
	}
```

{{% /tab %}}

{{% tab tabName="Complete Demo" %}}

```go
func TestDemo3(t *testing.T) {
	pipeline := ograph.NewPipeline()

	n1 := ograph.NewElement("N1").UseFn(func() error {
		fmt.Println("N1 running")
		return nil
	})
	n2 := ograph.NewElement("N2").UseFn(func() error {
		fmt.Println("N2 running")
		return nil
	})

	c := ograph.NewElement("C1").UseFactory(ogimpl.Choose, n1, n2).Params("ChooseExpr", "index")

	pipeline.Register(c)

	state := ograph.NewState()
	state.Set("index", 1) // choose node by index

	if err := pipeline.Run(context.TODO(), state); err != nil {
		t.Error(err)
	}
}
```

{{% /tab %}}
{{< /tabs >}}

```mermaid
---
title: Demo pipeline 3
---

flowchart LR
	subgraph C1
	direction LR

	C1_Start --> N1((N1))
	N1 --> C1_End

	C1_Start --x N2((N2))
	N2 --x C1_End
    end

	A[Start] --> C1
	C1 --> B[End]
```

This function is implemented through [Cluster](features/cluster.md), and other functions such as race execution are also implemented through `Cluster`.

## Use Node & Factory

The previous demo showed the simplest usage method. Further, you can use nodes or factories as pipeline elements to execute logic.

**Declare a node**
```go
type Printer struct {
	ograph.BaseNode //  Inherit the base node.
}

func (printer *Printer) Run(ctx context.Context, state ogcore.State) error {
	// Inherit the Name method from the base node.
	// Return the name set in the pipeline element (NewElement("N1")).
	fmt.Printf("[%s running]\n", printer.Name()) // [N1 running]

	// Get data from state by key.
	fmt.Println(state.Get("key"))
	return nil
}
```

**Use node in element**
```go
n1 := ograph.NewElement("N1").UseNode(&Printer{})
```

Using `UseFn` and `UseNode` is essentially a singleton pattern, and it is recommended to use the factory pattern in OGraph. More details is [here](features/factory.md).

**Declare a factory**

```go
pipeline.RegisterFactory("PrinterFactory", func() ogcore.Node {
		return &Printer{}
	})
```

**Use factory in element**

```go
n1 := ograph.NewElement("N1").UseFactory("PrinterFactory")
n2 := ograph.NewElement("N2").UseFactory("PrinterFactory")
```