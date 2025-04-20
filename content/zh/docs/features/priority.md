---
weight: 202
title: "优先级调度"
description: "关于如何在资源有限的情况下使节点分优先级调度"
icon: "article"
date: "2025-04-20T18:46:29+08:00"
lastmod: "2025-04-20T18:46:29+08:00"
draft: false
toc: true
---

在某些资源受限的情况下，可以使用 OGraph 提供的优先级调度功能，使得重要节点优先执行。

例如在图片处理程序中，需要对一批图片进行计算处理。这些图片大小不一，希望优先处理小图片，这样在短时间内可以处理尽可能多的图片（图片越小，处理越快）。

这种情况下就需要限制并发度，并且让小图片任务先执行。（在完全并发情况下，大图片任务会挤占小图片任务的CPU时间）

## 示例

```go
	pipeline := ograph.NewPipeline()

	n1 := ograph.NewElement("n1").UseFn(func() error {
		fmt.Println("n1 running")
		return nil
	}).SetPriority(99)

	n2 := ograph.NewElement("n2").UseFn(func() error {
		fmt.Println("n2 running")
		return nil
	}).SetPriority(1)

	pipeline.Register(n1).Register(n2)

	pipeline.ParallelismLimit = 1

	if err := pipeline.Run(context.Background(), nil); err != nil {
		fmt.Println(err)
	}
```

上面代码中用 `pipeline.ParallelismLimit = 1` 限制了并发度，如果不限制并发度，节点 n2 还是会马上执行。

建议对计算密集任务限制并发度为 CPU 核数，对于需要占用大量内存或者其他资源的任务，设置为总的资源量除于任务平均消耗量。

输出结果：

```
n1 running
n2 running
```