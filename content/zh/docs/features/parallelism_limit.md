---
weight: 202
title: "并发度限制"
description: "关于如何限制节点并发度"
icon: "article"
date: "2025-04-20T19:25:06+08:00"
lastmod: "2025-04-20T19:25:06+08:00"
draft: false
toc: true
---

在现实世界中，资源总是受限的。CPU 资源是受限的，内存容量是受限的，网络带宽是受限的。

所以如果 pipeline 节点并发度过高可能出现：

1. CPU：严重影响其他进程运行
2. 内存：OOM

OGraph 支持设置并发度限制来避免这些问题。

## 示例

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

输出结果：

```
n1 running
n1 stop
n2 running
n2 stop
```

从输出结果可以看到，虽然 n1 节点和 n2 节点之间没有依赖关系，但在设置并发度为一后，同一时间只能有一个节点在运行。因此即使 n1 节点陷入睡眠了 1 秒，期间 n2 也不能开始运行。

{{< alert context="info" text="这里说的并发度，更准确的说，应该是指并行度。" />}}