---
weight: 999
title: "Parallel Merge Sort"
description: ""
icon: "article"
date: "2025-05-04T22:58:38+08:00"
lastmod: "2025-05-04T22:58:38+08:00"
draft: false
toc: true
---

## Example Description

A primary use case for OGraph is accelerating parallel computations. Taking merge sort as an example, conventional merge sort algorithms do not leverage multi-core CPU parallelism.

During the merge sort process, each subsequence needs to be sorted. If these sorts can be performed concurrently, it significantly reduces computation time.

In this scenario, OGraph enables task parallelism without complicating the code structure.

## Example Code

The following code implements a parallelized merge sort. It first defines a merge function to combine two already sorted sequences.

Then, it defines sorting tasks for subsequences, merges them into the final result using the merge function, and defines a validation task to check the final result.

Finally, a pipeline is created, these tasks are registered, and the pipeline is executed.

(For code brevity, standard library sorting is used for subsequences rather than recursively applying the merge function as in conventional merge sort.)

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
