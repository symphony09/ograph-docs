---
weight: 301
title: "内置包装器"
description: "内置包装器的详细参考文档"
icon: "article"
date: "2025-03-09T21:26:35+08:00"
lastmod: "2025-03-09T21:26:35+08:00"
draft: false
toc: true
---

## 异步执行

> 用于异步执行被包装的节点

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
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

{{% tab tabName="参数说明" %}}
| 参数名 | 必需 | 含义 | 类型 | 示例 |
| :----- | :--- | :--- | ---- | :--- |
| -      | -    | -    | -    | -    |
{{% /tab %}}

{{% tab tabName="小贴士" %}}
在没有额外同步机制的情况下，无法保证能获取异步逻辑的执行结果。

为了避免不一致，异步节点对 `state`的读写对其他节点来说是透明的。也就是说，其他节点始终无法观察到异步节点对`state`的修改，而不是某些情况下能观察到，某些情况下观察不到。

如果某一个或者多个节点依赖异步节点的执行结果，应该考虑改为普通节点，或者使用额外的同步机制来传递结果。
{{% /tab %}}
{{< /tabs >}}

## 调试执行

> 用于把被包装节点的状态调试记录在日志中

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
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

{{% tab tabName="参数说明" %}}
| 参数名(Name) | 必需(Required) | 含义(Meaning) | 类型(Type) | 示例(Example) |
| :----------- | :------------- | :------------ | ---------- | :------------ |
| -            | -              | -             | -          | -             |
{{% /tab %}}

{{% tab tabName="小贴士" %}}
可以在 state 中传入 id 用于关联某次执行过程。

You can pass the `id` into the `state` to associate a specific execution process.

```go
	// ...  Declare pipeline and element

	state := ograph.NewState()
	ogimpl.SetTraceId(state, "your_trace_id")

	p.Register(e).Run(context.TODO(), state)
```
{{% /tab %}}
{{< /tabs >}}

## 延迟执行

> 用于延迟执行被包装的节点

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
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

{{% tab tabName="参数说明" %}}
| 参数名(Name) | 必需(Required) | 含义(Meaning) | 类型(Type)    | 示例(Example)                       |
| :----------- | :------------- | :------------ | ------------- | :---------------------------------- |
| Wait         | ✗              | wait duration | string        | "1h59m59s"                          |
| Wait         | ✗              | wait duration | time.Duration | time.Second                         |
| Until        | ✗              | wait until    | string        | "2024-05-25T10:44:57.6061908+08:00" |
| Until        | ✗              | wait until    | time.Time     | time.Time{}                         |

Wait 和 Until 支持两种类型的参数，使用 string 类型作为 Until 参数时，需要符合 time.RFC3339Nano 格式。
{{% /tab %}}

{{% tab tabName="小贴士" %}}
可以同时设置 Wait 和 Until 参数，都满足时执行被包装节点
{{% /tab %}}
{{< /tabs >}}

## 循环执行

> 用于循环执行被包装的节点

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
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

{{% tab tabName="参数说明" %}}
| 参数名(Name) | 必需(Required) | 含义(Meaning) | 类型(Type)    | 示例(Example) |
| :----------- | :------------- | :------------ | ------------- | :------------ |
| LoopTimes    | ✔              | 循环次数      | int           | 3             |
| LoopInterval | ✗              | 循环间隔      | string        | "1s"          |
| LoopInterval | ✗              | 循环间隔      | time.Duration | time.Second   |

LoopInterval 支持两种类型的参数
{{% /tab %}}

{{% tab tabName="小贴士" %}}

{{% /tab %}}
{{< /tabs >}}

## 错误重试

> 用于重试运行错误的节点

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
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

{{% tab tabName="参数说明" %}}
| 参数名(Name)  | 必需(Required) | 含义(Meaning) | 类型(Type)    | 示例(Example) |
| :------------ | :------------- | :------------ | ------------- | :------------ |
| MaxRetryTimes | ✗              | 最大重试次数  | int           | 3             |
| RetryDelay    | ✗              | 重试延迟      | string        | "1s"          |
| RetryDelay    | ✗              | 重试延迟      | time.Duration | time.Second   |

如果 MaxRetryTimes 小于或等于 0，则使用默认值 1。
{{% /tab %}}

{{% tab tabName="小贴士" %}}

{{% /tab %}}
{{< /tabs >}}

## 静默执行

> 用于忽略被包装的节点的运行错误

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
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

{{% tab tabName="参数说明" %}}
| 参数名(Name) | 必需(Required) | 含义(Meaning) | 类型(Type) | 示例(Example) |
| :----------- | :------------- | :------------ | ---------- | :------------ |
| -            | -              | -             | -          | -             |
{{% /tab %}}

{{% tab tabName="小贴士" %}}
节点报错会直接导致 pipeline 停止运行并返回错误，所以有时需要忽略非关键节点报错，报错信息仍然可以通过日志追踪到。
{{% /tab %}}
{{< /tabs >}}

## 超时限制执行

> 用于超时取消被包装的节点

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
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

{{% tab tabName="参数说明" %}}
| 参数名(Name) | 必需(Required) | 含义(Meaning) | 类型(Type)    | 示例(Example) |
| :----------- | :------------- | :------------ | ------------- | :------------ |
| Timeout      | ✔              | 超时时间      | string        | "1s"          |
| Timeout      | ✔              | 超时时间      | time.Duration | time.Second   |


Timeout 支持两种类型的参数
{{% /tab %}}

{{% tab tabName="小贴士" %}}
1. Timeout Wrapper 并不能控制节点停止运行和释放资源，只是通过 ctx 传递取消信号。同时向上报告超时错误，使 pipeline 可以不再等待节点运行结束。

2. 在超时后，超时节点对state的写操作（set，update）将会失效。

3. 如果不希望超时错误影响 pipeline 继续执行，可以配合 Silent Wrapper 一起使用。

{{% /tab %}}
{{< /tabs >}}

## 追踪执行

> 用于把被包装节点的执行过程记录在日志中

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
```go
	p := ograph.NewPipeline()

	e := ograph.NewElement("node_to_be_trace").
		UseNode(&ograph.BaseNode{}).
		Wrap(ogimpl.Trace)

	// The execution process of the node will be recorded in the log.
	p.Register(e).Run(context.TODO(), nil)
```
{{% /tab %}}

{{% tab tabName="参数说明" %}}
| 参数名(Name) | 必需(Required) | 含义(Meaning) | 类型(Type) | 示例(Example) |
| :----------- | :------------- | :------------ | ---------- | :------------ |
| -            | -              | -             | -          | -             |

{{% /tab %}}

{{% tab tabName="小贴士" %}}
可以在 state 中传入 id 用于关联某次执行过程。

```go
	// ...  Declare pipeline and element

	state := ograph.NewState()
	ogimpl.SetTraceId(state, "your_trace_id")

	p.Register(e).Run(context.TODO(), state)
```
{{% /tab %}}
{{< /tabs >}}


