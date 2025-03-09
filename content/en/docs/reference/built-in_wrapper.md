---
weight: 301
title: "Built-in Wrapper"
description: "built-in wrappers reference"
icon: "article"
date: "2025-02-26T21:08:19+08:00"
lastmod: "2025-02-26T21:08:19+08:00"
draft: false
toc: true
---

## Async Wrapper

> Used for wrapping asynchronous nodes for concurrent execution

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
	stopCh := make(chan struct{})

	p := ograph.NewPipeline()

	e := ograph.NewElement("node_to_async_run").
		UseFn(func() error {
			time.Sleep(time.Second)
			fmt.Println("node end")
			stopCh <- struct{}{}
			return nil
		}).
		Wrap(ogimpl.Async)

	// async node wont't block pipeline
	p.Register(e).Run(context.TODO(), nil)

	// The pipeline will end immediately after completion,
	// while the node will end one second after startup.
	fmt.Println("pipeline end")
	<-stopCh
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name | Required | Meaning | Type | Example |
| :--- | :------- | :------ | ---- | :------ |
| -    | -        | -       | -    | -       |

{{% /tab %}}

{{% tab tabName="Tips" %}}
Without additional synchronization mechanisms, it is not guaranteed to obtain the result of the execution of asynchronous logic.

To avoid inconsistencies, the read-write operations on `state` by asynchronous nodes are transparent to other nodes. This means that other nodes cannot observe the modifications made by asynchronous nodes to `state`, not in some cases, but in all cases.

If a node or multiple nodes depend on the execution result of an asynchronous node, it is recommended to consider changing to a regular node or using an additional synchronization mechanism to transmit the result.
{{% /tab %}}
{{< /tabs >}}

## Debug Wrapper

> The state of the wrapped node can be debugged and recorded in the log.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
type ReadWriteStateNode struct {
	ograph.BaseNode
}

func (node *ReadWriteStateNode) Run(ctx context.Context, state ogcore.State) error {
	state.Set("i", 1)
	state.Update("i", func(val any) any {
		return "1"
	})
	state.Get("i")
	return nil
}

```

```go
	p := ograph.NewPipeline()

	e := ograph.NewElement("node_to_debug").
		UseNode(&ReadWriteStateNode{}).
		Wrap(ogimpl.Debug)

	// The state of a node before and after reading and writing will be recorded in the log.
	p.Register(e).Run(context.TODO(), nil)
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name | Required | Meaning | Type | Example |
| :--- | :------- | :------ | ---- | :------ |
| -    | -        | -       | -    | -       |

{{% /tab %}}

{{% tab tabName="Tips" %}}
You can pass the `id` into the `state` to associate a specific execution process.

```go
	// ...  Declare pipeline and element

	state := ograph.NewState()
	ogimpl.SetTraceId(state, "your_trace_id")

	p.Register(e).Run(context.TODO(), state)
```
{{% /tab %}}
{{< /tabs >}}

## Delay Wrapper

> Used for delaying the execution of wrapped nodes.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
	p := ograph.NewPipeline()

	e := ograph.NewElement("node_to_delay").
		UseFn(func() error {
			fmt.Println("node start running")
			return nil
		}).
		Wrap(ogimpl.Delay).
		Params("Wait", "3s")

	// node_to_delay will run after 3 seconds
	p.Register(e).Run(context.TODO(), nil)
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name  | Required | Meaning       | Type          | Example                             |
| :---- | :------- | :------------ | ------------- | :---------------------------------- |
| Wait  | ✗        | wait duration | string        | "1h59m59s"                          |
| Wait  | ✗        | wait duration | time.Duration | time.Second                         |
| Until | ✗        | wait until    | string        | "2024-05-25T10:44:57.6061908+08:00" |
| Until | ✗        | wait until    | time.Time     | time.Time{}                         |

"Wait" and "Until" support two types of parameters. When using the "Until" parameter with a string type, it must adhere to the time.RFC3339Nano format.

{{% /tab %}}

{{% tab tabName="Tips" %}}
You can set both Wait and Until parameters simultaneously, and execute the wrapped node when both conditions are met.
{{% /tab %}}
{{< /tabs >}}

## Loop Wrapper

>  Used for looping execution of wrapped nodes.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
	p := ograph.NewPipeline()

	e := ograph.NewElement("node_to_loop").
		UseFn(func() error {
			fmt.Println("node start running")
			return nil
		}).
		Wrap(ogimpl.Loop).
		Params("LoopTimes", 3)

	// node_to_loop will loop run 3 times
	p.Register(e).Run(context.TODO(), nil)
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name         | Required | Meaning       | Type          | Example     |
| :----------- | :------- | :------------ | ------------- | :---------- |
| LoopTimes    | ✔        | loop times    | int           | 3           |
| LoopInterval | ✗        | loop interval | string        | "1s"        |
| LoopInterval | ✗        | loop interval | time.Duration | time.Second |

LoopInterval support two types of parameters.

{{% /tab %}}

{{% tab tabName="Tips" %}}

{{% /tab %}}
{{< /tabs >}}

## Retry Wrapper

> For retrying failed nodes.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
	var once sync.Once

	p := ograph.NewPipeline()

	e := ograph.NewElement("problem_node").
		UseFn(func() error {
			var err error

			fmt.Println("node start running")

			once.Do(func() { // Returns error only once.
				err = errors.New("something going wrong")
			})

			return err
		}).
		Wrap(ogimpl.Retry)

	// The pipeline completes normally after the retry fails.
	err := p.Register(e).Run(context.TODO(), nil)
	fmt.Println(err == nil)
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name          | Required | Meaning         | Type          | Example     |
| :------------ | :------- | :-------------- | ------------- | :---------- |
| MaxRetryTimes | ✗        | max retry times | int           | 3           |
| RetryDelay    | ✗        | retry delay     | string        | "1s"        |
| RetryDelay    | ✗        | retry delay     | time.Duration | time.Second |

If MaxRetryTimes is less than or equal to 0, use the default value of 1.

{{% /tab %}}

{{% tab tabName="Tips" %}}

{{% /tab %}}
{{< /tabs >}}

## Silent Wrapper

>  Ignore running errors of wrapped nodes.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
	p := ograph.NewPipeline()

	e := ograph.NewElement("problem_node").
		UseFn(func() error {
			return errors.New("something going wrong")
		}).
		Wrap(ogimpl.Silent)

	// Pipeline will not fail due to failed nodes in the problem.
	err := p.Register(e).Run(context.TODO(), nil)
	fmt.Println(err == nil)
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name | Required | Meaning | Type | Example |
| :--- | :------- | :------ | ---- | :------ |
| -    | -        | -       | -    | -       |

{{% /tab %}}

{{% tab tabName="Tips" %}}
Nodes reporting errors will directly cause the pipeline to stop running and return an error. Therefore, sometimes it is necessary to ignore non-critical node errors, and error information can still be tracked through logs.
{{% /tab %}}
{{< /tabs >}}

## Timeout Wrapper

> Cancel wrapped nodes that exceed a timeout.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
	p := ograph.NewPipeline()

	e := ograph.NewElement("slow_node").
		UseFn(func() error {
			time.Sleep(time.Minute)
			return nil
		}).
		Wrap(ogimpl.Timeout).
		Params("Timeout", "10ms")

	// After a node times out, the pipeline stops running and returns a timeout error.
	err := p.Register(e).Run(context.TODO(), nil)
	fmt.Println(errors.Is(err, ogimpl.ErrTimeout))
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name    | Required | Meaning | Type          | Example     |
| :------ | :------- | :------ | ------------- | :---------- |
| Timeout | ✔        | timeout | string        | "1s"        |
| Timeout | ✔        | timeout | time.Duration | time.Second |

Timeout support two types of parameters.

{{% /tab %}}

{{% tab tabName="Tips" %}}
1. The Timeout Wrapper does not control node termination and resource release, but instead passes a cancel signal through ctx. It also reports a timeout error to allow the pipeline to proceed without waiting for the node.

2. After timeout, the failed timeout node's write operation (set, update) to state will fail.

3. To avoid allowing timeout errors to affect the pipeline's continued execution, you can use the Silent Wrapper in conjunction with the Timeout Wrapper.
{{% /tab %}}
{{< /tabs >}}

## Trace Wrapper

> It is used to record the execution process of the wrapped node in the log.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
	p := ograph.NewPipeline()

	e := ograph.NewElement("node_to_be_trace").
		UseNode(&ograph.BaseNode{}).
		Wrap(ogimpl.Trace)

	// The execution process of the node will be recorded in the log.
	p.Register(e).Run(context.TODO(), nil)
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name | Required | Meaning | Type | Example |
| :--- | :------- | :------ | ---- | :------ |
| -    | -        | -       | -    | -       |

{{% /tab %}}

{{% tab tabName="Tips" %}}
You can pass the `id` into the `state` to associate a specific execution process.

```go
	// ...  Declare pipeline and element

	state := ograph.NewState()
	ogimpl.SetTraceId(state, "your_trace_id")

	p.Register(e).Run(context.TODO(), state)
```
{{% /tab %}}
{{< /tabs >}}