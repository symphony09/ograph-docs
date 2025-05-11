---
weight: 202
title: "Priority Scheduling"
description: "How to prioritize node execution when resources are limited"
icon: "article"
date: "2025-04-20T18:46:29+08:00"
lastmod: "2025-04-20T18:46:29+08:00"
draft: false
toc: true
---

In resource-constrained scenarios, OGraph's priority scheduling feature allows important nodes to execute first.

For example, in an image processing application that needs to process a batch of images of varying sizes, smaller images should be prioritized to process as many images as possible in a short time (smaller images take less time to process).

In this scenario, it's necessary to limit concurrency and let smaller image tasks execute first. (In fully concurrent scenarios, large image tasks would consume CPU time from smaller tasks)

## Example

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

In the code above, `pipeline.ParallelismLimit = 1` limits concurrency. Without this limit, node n2 would still execute immediately.

It's recommended to set concurrency limits for compute-intensive tasks to the number of CPU cores, and for tasks requiring significant memory or other resources, set it to total resources divided by average resource consumption per task.

Output:

```
n1 running
n2 running
```
