---
weight: 202
title: "Pause, Continue, and Cancel"
description: "How to pause, continue, and cancel a pipeline"
icon: "article"
date: "2025-04-20T15:22:48+08:00"
lastmod: "2025-04-20T15:22:48+08:00"
draft: false
toc: true
---

## Example

OGraph Pipeline supports pausing, resuming, and canceling execution. Here's an example:

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

	pause() // pause before running node b

	time.Sleep(time.Second)

	continueRun() // b continues running, after 1.5s (0.5+1), not 1s

	time.Sleep(time.Millisecond * 500)

	cancel() // cancel before running node c, so node c will never execute

	err := wait() // wait for pipeline completion

	fmt.Println(err) // should get context canceled error
```

## Execution Process

The example code execution flow is as follows:

```mermaid
%%{init: { 'logLevel': 'debug', 'theme': 'default', 'timeline': {'disableMulticolor': true} } }%%
timeline
    title Execution process
    0s~0.5s : Node a running
    0.5s~1s : Node a finishes
         : Pipeline pauses
    1s~1.5s : Pipeline paused
    1.5s~2s : Pipeline continues : Node b running
    2s~2.5s : Node b finishes : Pipeline canceled
```

{{< alert context="info" text="Pausing does not affect nodes already running. If a node does not support cancellation, cancellation will not affect nodes already running." />}}
