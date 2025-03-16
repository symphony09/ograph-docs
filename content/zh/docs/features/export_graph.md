---
weight: 202
title: "导出图&导入图"
description: "关于如何导出图和导入图的指南"
icon: "article"
date: "2025-03-16T23:04:29+08:00"
lastmod: "2025-03-16T23:04:29+08:00"
draft: false
toc: true
---

## 使用场景

在 OGraph 中，你可以将一个 pipeline 的依赖关系图导出，并在另一个 pipeline 中导入执行。如果你需要在不同地方构建和执行 pipeline，那么图导出功能可以帮你做到这一点

## 例子

下面是一个简单的例子

```go
	pipeline := ograph.NewPipeline()

	amy := ograph.NewElement("Amy").UseFactory("Person").Wrap(ogimpl.Trace)
	jack := ograph.NewElement("Jack").UseFactory("Person").Wrap(ogimpl.Trace)

	pipeline.Register(amy).
		Register(jack, ograph.Rely(amy))

	if graphData, err := pipeline.DumpGraph(); err != nil {
		t.Error(err)
	} else {
		fmt.Println(string(graphData))

		newPipeline := ograph.NewPipeline()
		newPipeline.LoadGraph(graphData)
		newPipeline.RegisterFactory("Person", func() ogcore.Node { return &ograph.BaseNode{} })

		if err := newPipeline.Run(context.TODO(), nil); err != nil {
			fmt.Println(err)
		}
	}
```

整个过程分为以下几步

1. 构建一个 pipeline
2. 将pipeline的图信息以 Json 格式导出，你可以把 Json 传输到其他地方
3. 新建 newPipeline，并导入上一步得到的图信息，然后执行

**输出**

```
{"Vertices":{"Amy":{"Name":"Amy","FactoryName":"Person","Wrappers":["Trace"]},"Jack":{"Name":"Jack","FactoryName":"Person","Wrappers":["Trace"]}},"Edges":[["Amy","Jack"]]}
2025/03/16 23:24:03 INFO node start NodeName=Amy TraceID=""
2025/03/16 23:24:03 INFO node finish NodeName=Amy TimeCost=12.0252ms TraceID=""
2025/03/16 23:24:03 INFO node start NodeName=Jack TraceID=""
2025/03/16 23:24:03 INFO node finish NodeName=Jack TimeCost=0s TraceID=""
```

{{< alert context="info" text="要导出的 pipeline 只能包含由工厂模式构建的 node 或者子 pipeline" />}}