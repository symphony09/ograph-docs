---
weight: 401
title: "MapReduce"
description: ""
icon: "article"
date: "2025-03-12T22:39:37+08:00"
lastmod: "2025-03-12T22:39:37+08:00"
draft: false
toc: true
---

## 样例说明

以下Go代码实现了一个简单的MapReduce流程，用于统计指定文本文件集中不同词汇的频率，并以字典的形式输出结果。具体来说，它定义了两个关键的逻辑组件：

1. **Mapper (映射器)**:
   - `Mapper`结构体中的`Run`方法首先尝试打开指定的输入文件（如“input1.txt”或“input2.txt”）。
   - 对于文件内容，代码逐行读取并分割单词。每个单词被视为一个单独的数据项。
   - 对于每个单词，它将作为键，其频率作为值存储在一个临时状态中。

2. **Reducer (归约器)**:
   - `Reducer`结构体中的`Run`方法接收来自所有映射操作的结果（即多个Mapper输出）并计算每个单词在整个数据集中的总频率。
   - 它汇总了之前由不同Mapper实例生成的各个词频，并将这些结果合并在一起，从而得到整个输入文件中每个单词的总数。

3. **管道处理**:
   - 使用`pipeline.Register()`函数来注册不同的Mapper和Reducer组件。代码创建了两个映射器（分别读取不同的输入文件），并将这两个Mapper的结果汇总到一个归约器实例进行最终处理。
   - 运行整个流程后，程序打印出每个单词及其在所有输入文件中出现的总次数。

## 样例代码

```go
package main

import (
	"bufio"
	"context"
	"fmt"
	"os"
	"strings"

	"github.com/symphony09/ograph"
	"github.com/symphony09/ograph/ogcore"
)

type Mapper struct {
	ograph.BaseNode

	InputFileName string
}

func (mapper *Mapper) Run(ctx context.Context, state ogcore.State) error {
	file, err := os.Open(mapper.InputFileName)
	if err != nil {
		return fmt.Errorf("Error opening file: %w", err)
	}
	defer file.Close()

	// 使用 bufio.Scanner 逐行读取文件内容
	scanner := bufio.NewScanner(file)

	// Map 阶段，将每行数据分割为键值对
	var mappedResults []string
	for scanner.Scan() {
		line := scanner.Text()

		var result []string
		for _, word := range strings.Fields(line) {
			result = append(result, fmt.Sprintf("%s\t%d", word, 1)) // 键值对：<word 1>
		}

		mappedResults = append(mappedResults, result...)
	}

	if err := scanner.Err(); err != nil {
		return fmt.Errorf("Error during mapping: %w", err)
	}

	// 将所有键值对添加到 map 中，以便进行 reduce 操作
	state.Update("map_results", func(val any) any {
		var mapResults map[string][]int
		if val == nil {
			mapResults = make(map[string][]int)
		} else {
			mapResults = val.(map[string][]int)
		}

		for _, m := range mappedResults {
			parts := strings.Split(m, "\t")
			word := parts[0]
			value := 1 // 在 Map 阶段生成的每个 <key, value> 对中，value 总是 1。
			mapResults[word] = append(mapResults[word], value)
		}

		return mapResults
	})

	return nil
}

type Reducer struct {
	ograph.BaseNode
}

func (reducer *Reducer) Run(ctx context.Context, state ogcore.State) error {
	results, _ := state.Get("map_results")
	mapResults := results.(map[string][]int)

	// Reduce 阶段，汇总所有相同键的值
	reduceResults := make(map[string]int)
	for word, values := range mapResults {
		sum := 0
		for _, value := range values {
			sum += value
		}

		reduceResults[word] = sum
	}

	state.Set("reduce_results", reduceResults)
	return nil
}

func main() {
	pipeline := ograph.NewPipeline()

	mapper1 := ograph.NewElement("M1").UseNode(&Mapper{InputFileName: "input1.txt"})
	mapper2 := ograph.NewElement("M2").UseNode(&Mapper{InputFileName: "input2.txt"})
	reducer := ograph.NewElement("R1").UseNode(&Reducer{})

	pipeline.Register(mapper1).
		Register(mapper2).
		Register(reducer, ograph.Rely(mapper1, mapper2))

	state := ograph.NewState()

	if err := pipeline.Run(context.Background(), state); err != nil {
		fmt.Println(err)
	} else {
		fmt.Println(state.Get("reduce_results"))
	}
}

```