---
weight: 202
title: "Export graph & import graph"
description: "A guide to the feature of export graph & import graph."
icon: "article"
date: "2025-03-16T23:37:49+08:00"
lastmod: "2025-03-16T23:37:49+08:00"
draft: false
toc: true
---


## Usage Scenario
In OGraph, you can export the dependency graph of a pipeline and import it for execution in another pipeline. If you need to build and execute pipelines in different locations, the graph export feature can help you achieve this.

## Example
Below is a simple example:

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


The entire process is divided into the following steps:

1. Build a pipeline.
2. Export the graph information of the pipeline in JSON format, and you can transfer the JSON to another location.
3. Create a new pipeline (newPipeline) and import the graph information from the previous step, then execute it.


**output**

```
{"Vertices":{"Amy":{"Name":"Amy","FactoryName":"Person","Wrappers":["Trace"]},"Jack":{"Name":"Jack","FactoryName":"Person","Wrappers":["Trace"]}},"Edges":[["Amy","Jack"]]}
2025/03/16 23:24:03 INFO node start NodeName=Amy TraceID=""
2025/03/16 23:24:03 INFO node finish NodeName=Amy TimeCost=12.0252ms TraceID=""
2025/03/16 23:24:03 INFO node start NodeName=Jack TraceID=""
2025/03/16 23:24:03 INFO node finish NodeName=Jack TimeCost=0s TraceID=""
```

{{< alert context="info" text="The pipeline to be exported can only contain nodes or sub-pipelines built using the factory pattern." />}}