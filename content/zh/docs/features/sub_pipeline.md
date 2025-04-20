---
weight: 201
title: "子 Pipeline"
description: "关于如何嵌套子 pipeline"
icon: "article"
date: "2025-04-20T18:19:09+08:00"
lastmod: "2025-04-20T18:19:09+08:00"
draft: false
toc: true
---

OGraph 支持嵌套子 Pipeline 来分治复杂的工作流，并且对嵌套深度没有限制。

## 示例

实现嵌套 pipeline 是非常简单的，只要声明一个 pipeline，然后像普通节点一样在另一个 pipeline 中注册它就可以了。

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

上面代码中，创建了两个子 pipeline，模拟两类事情，然后再在主 pipeline 中注册为两个节点，并指定依赖关系。

为了代码简洁，构建子pipeline的逻辑放到了一个函数中，实际上和一般的创建过程没有任何区别

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

输出结果：

```
->LearnEnglish->LearnProgramming->Sleep->PlayGame
```

子pipeline内部节点没有指定依赖关系，随机顺序打印，主 pipeline 指定了子 pipeline 的依赖关系，以固定顺序打印。