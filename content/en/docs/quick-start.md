---
weight: 101
title: "Quick Start"
description: "A guide to getting up and running a pipeline."
icon: "rocket_launch"
date: "2025-02-22T21:19:07+08:00"
lastmod: "2025-02-22T21:19:07+08:00"
draft: false
toc: true
---

## Install

```shell
go get github.com/symphony09/ograph
```

## Demo

### Step.1 Declare a node

```go
type Person struct {
	ograph.BaseNode
}

func (person *Person) Run(ctx context.Context, state ogcore.State) error {
	fmt.Printf("Hello, i am %s.\n", person.Name())
	return nil
}
```

### Step.2 Create and run pipeline

```go
func TestHello(t *testing.T) {
	pipeline := ograph.NewPipeline()

	zhangSan := ograph.NewElement("ZhangSan").UseNode(&Person{})
	liSi := ograph.NewElement("LiSi").UseNode(&Person{})

	pipeline.Register(zhangSan).
		Register(liSi, ograph.Rely(zhangSan))

	if err := pipeline.Run(context.TODO(), nil); err != nil {
		t.Error(err)
	}
}
```

**Output**

```
    Hello, i am ZhangSan.
    Hello, i am LiSi.
```