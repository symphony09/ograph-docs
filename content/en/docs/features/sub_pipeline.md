---
weight: 201
title: "Sub Pipeline"
description: "How to Nest Sub Pipelines"
icon: "article"
date: "2025-04-20T18:19:09+08:00"
lastmod: "2025-04-20T18:19:09+08:00"
draft: false
toc: true
---

OGraph supports nested sub pipelines to divide and conquer complex workflows, with no limit on nesting depth.

## Example

Implementing nested pipelines is simple. Just declare a pipeline and register it in another pipeline as you would with a regular node.

```go
	study := newSubPipeline("LearnProgramming", "LearnEnglish")
	relax := newSubPipeline("PlayGame", "Sleep")

	studyThings := ograph.NewElement("StudyThings").UseNode(study)
	relaxThings := ograph.NewElement("RelaxThings").UseNode(relax)

	day := ograph.NewPipeline()
	day.Register(studyThings).Register(relaxThings, ograph.Rely(studyThings))

	if err := day.Run(context.TODO(), nil); err != nil {
		fmt.Println(err)
	}
```

In the code above, two sub pipelines are created to simulate two types of tasks. These are then registered as two nodes in the main pipeline with specified dependency relationships.

To keep the code concise, the logic for building the subpipeline is placed in a function. This is actually no different from the general creation process.

```go
func newSubPipeline(things ...string) *ograph.Pipeline {
	pipeline := ograph.NewPipeline()

	for _, thing := range things {
		thing := thing
		pipeline.Register(ograph.NewElement(thing).UseFn(func() error {
			fmt.Printf("->%s", thing)
			return nil
		}))
	}

	return pipeline
}
```

Output result:

```
->LearnEnglish->LearnProgramming->Sleep->PlayGame
```

The nodes inside the subpipeline do not specify dependency relationships and are printed in random order. The main pipeline specifies the dependency relationships between sub pipelines, resulting in a fixed order of printing.
