---
weight: 202
title: "Parallelism Limit"
description: "How to limit node parallelism"
icon: "article"
date: "2025-04-20T19:25:06+08:00"
lastmod: "2025-04-20T19:25:06+08:00"
draft: false
toc: true
---

In the real world, resources are always limited. CPU resources are limited, memory capacity is limited, and network bandwidth is limited.

If the pipeline node parallelism is too high, it may lead to:

1. CPU: Severely impact other processes
2. Memory: Out Of Memory (OOM)

OGraph supports setting parallelism limits to avoid these issues.

## Example

```go
	pipeline := ograph.NewPipeline()

	n1 := ograph.NewElement("n1").UseFn(func() error {
		fmt.Println("n1 running")
		time.Sleep(time.Second)
		fmt.Println("n1 stop")
		return nil
	})

	n2 := ograph.NewElement("n2").UseFn(func() error {
		fmt.Println("n2 running")
		fmt.Println("n2 stop")
		return nil
	})
	pipeline.Register(n1).Register(n2)

	pipeline.ParallelismLimit = 1

	if err := pipeline.Run(context.Background(), nil); err != nil {
		fmt.Println(err)
	}
```

Output result:

```
n1 running
n1 stop
n2 running
n2 stop
```

From the output, you can see that although there's no dependency between n1 and n2 nodes, after setting the parallelism to 1, only one node can run at a time. Therefore, even if n1 node sleeps for 1 second, n2 cannot start during that period.

{{< alert context="info" text="What's mentioned here as 'parallelism' is more accurately referred to as 'concurrency' in technical terms." />}}
