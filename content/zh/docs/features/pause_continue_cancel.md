---
weight: 202
title: "暂停、继续和取消"
description: "关于如何暂停、继续和取消 pipeline"
icon: "article"
date: "2025-04-20T15:22:48+08:00"
lastmod: "2025-04-20T15:22:48+08:00"
draft: false
toc: true
---

## 示例

OGraph Pipeline 支持暂停运行、继续运行和取消运行，示例代码如下：

```go
	pipeline := ograph.NewPipeline()

	var startTime time.Time

	a := ograph.NewElement("a").UseFn(func() error {
		fmt.Println("a running")
		startTime = time.Now()
		time.Sleep(time.Second)
		return nil
	})

	b := ograph.NewElement("b").UseFn(func() error {
		fmt.Println("b running, after", time.Since(startTime))
		time.Sleep(time.Second)
		return nil
	})

	c := ograph.NewElement("c").UseFn(func() error {
		fmt.Println("c running, after", time.Since(startTime))
		return nil
	})

	pipeline.Register(a).Register(b, ograph.Rely(a)).Register(c, ograph.Rely(b))

	ctx, cancel := context.WithCancel(context.Background())

	pause, continueRun, wait := pipeline.AsyncRun(ctx, nil)

	time.Sleep(time.Millisecond * 500)

	pause() // pause before run node b

	time.Sleep(time.Second)

	continueRun() // b continue run, after 1.5s (0.5+1), not 1s

	time.Sleep(time.Millisecond * 500)

	cancel() // cancel before run node c, so node c will never be ran

	err := wait() // wait pipeline finish

	fmt.Println(err) // should get context canceled err
```

## 流程图

示例代码运行过程如下：

```mermaid
%%{init: { 'logLevel': 'debug', 'theme': 'default', 'timeline': {'disableMulticolor': true} } }%%
timeline
    title Execution process
    0s~0.5s : Node a running
    0.5s~1s : Node a finish
         : Pipeline pause
    1s~1.5s : Pipeline pause
    1.5s~2s : Pipeline continue : Node b running
    2s~2.5s : Node b finish : Pipeline cancel

```

{{< alert context="info" text="暂停不会影响正在运行中的节点。如果节点本身不支持取消，那么取消也不会影响运行中的节点。" />}}