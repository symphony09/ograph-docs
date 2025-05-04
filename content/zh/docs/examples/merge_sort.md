---
weight: 999
title: "并行合并排序"
description: ""
icon: "article"
date: "2025-05-04T22:58:38+08:00"
lastmod: "2025-05-04T22:58:38+08:00"
draft: false
toc: true
---

## 样例说明

一个 OGraph 的主要使用场景是并行计算加速。以合并排序为例，常见的合并排序算法没有利用多核 CPU 并行计算。

在合并排序的过程，需要对每个子序列进行排序，如果每个子序列的排序可以同时进行，那么就极大的的缩短计算时间。

在这种情况下，OGraph 可以让计算任务并行化，并且不让代码变成一团乱麻。

## 样例代码

以下代码实现了并行化的合并排序，代码中首先定义了一个合并函数用于合并两个已经排序完成的序列。

然后定义了一些排序任务，用于对子序列进行排序，并用合并函数合并到最终结果中。还定义了一个检查任务，用于对最终结果进行检验。

最后新建一个pipeline，注册这些任务并执行。

（为了代码简洁，对子序列的排序直接使用了标准库，而不是像常规合并排序一样递归地使用合并函数。）

```go
package main

import (
	"context"
	"fmt"
	"log"
	"math/rand"
	"slices"
	"sync"

	"github.com/symphony09/ograph"
)

func merge(left, right []int) []int {
	result := make([]int, 0, len(left)+len(right))
	for len(left) > 0 || len(right) > 0 {
		if len(left) == 0 {
			return append(result, right...)
		}
		if len(right) == 0 {
			return append(result, left...)
		}
		if left[0] < right[0] {
			result = append(result, left[0])
			left = left[1:]
		} else {
			result = append(result, right[0])
			right = right[1:]
		}
	}
	return result
}

func main() {
	size := 1000
	randomInts := make([]int, 0, size)
	sortedInts := make([]int, 0, size)
	mux := &sync.Mutex{}

	for i := 0; i < size; i++ {
		randomInts = append(randomInts, rand.Intn(10000))
	}

	sortTasks := make([]*ograph.Element, 0, 10)

	for i := 0; i < 10; i++ {
		part := randomInts[i*100 : (i+1)*100]

		t := ograph.NewElement(fmt.Sprintf("t%d", i)).UseFn(func() error {
			slices.Sort(part)
			mux.Lock()
			sortedInts = merge(sortedInts, part)
			mux.Unlock()
			return nil
		})

		sortTasks = append(sortTasks, t)
	}

	checkTask := ograph.NewElement("check").UseFn(func() error {
		if len(sortedInts) != 1000 || !slices.IsSorted(sortedInts) {
			return fmt.Errorf("got wrong sort result")
		}
		return nil
	})

	p := ograph.NewPipeline()
	err := p.Register(checkTask, ograph.Rely(sortTasks...)).Run(context.Background(), nil)
	if err != nil {
		log.Fatalf("run merge sort pipeline failed, err: %v\n", err)
	}

	fmt.Println("result:", sortedInts)
}
```