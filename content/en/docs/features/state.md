---
weight: 201
title: "State"
description: ""
icon: "article"
date: "2025-02-26T21:14:03+08:00"
lastmod: "2025-02-26T21:14:03+08:00"
draft: false
toc: true
---

In ograph, nodes share data through State. Specifically, State is a data container that is passed as a parameter when a node runs, allowing nodes to read and write their own data, as well as to access data written by other nodes.

## How to use

`State` is one of the running parameters of a node, which can call corresponding methods for read and write operations during runtime, as shown in the following example code:

```go
func (printer *Printer) Run(ctx context.Context, state ogcore.State) error {
	// Get data from state by key.
	fmt.Println(state.Get("text"))
}
```

## Interface definition

In order to meet the needs of different scenarios, the `State` is designed as an interface. The interface definition is as follows:

```go
type State interface {
	Get(key any) (any, bool)
	Set(key any, val any)
	Update(key any, updateFunc func(val any) any)
}
```

As shown in the code, the `State` interface contains three methods:

1. **Get**: Used for reading data, with the parameter being the key corresponding to the data, the first return value is the value of the data, and if the value does not exist, the second return value should be `false`.
2. **Set**: Used for writing data, with the first parameter being the key and the second parameter being the value.
3. **Update**: Similar to `Set`, used for updating data. The first parameter is the key, and the second parameter is a function that receives the old value and returns the new value.



## Default State

In most cases, you don't need to implement a `State` yourself. You can get the default `State` by using the following code.

```go
// declare
state := ograph.NewState()

// init
state.Set("param_key", "param_val")

// pass it to pipeline
pipeline.Run(context.TODO(), state)
```

You can also pass `nil` directly to the pipeline, in this case, the pipeline will automatically create a default `State` for the node to use.

{{< alert context="info" text="The default `State` will enable lock protection when writing data, to ensure concurrency safety. Please be aware of this to avoid blocking." />}}

## Custom State

As previously mentioned, in most cases, you don't need to customize implement a `State`. However, the default implementation may not support all business scenarios. In the following situations, you may need to customize `State`.

1. Higher performance. The default `State` uses read-write locks to ensure concurrency safety, which sacrifices performance to some extent. If you implement concurrency safety using other methods, you can replace the default implementation with an unlock `State`.
2. Business requirements. For example, **persistent functionality**, **read-write intercept functionality**, **remote read-write functionality** (e.g., `redis`, `etcd`), etc.

When implementing custom `State`, it is recommended to use a composition approach, such as:

```go
type PrintSetState struct {
	innerState ogcore.State
}

func (state *PrintSetState) Set(key any, val any) {
	fmt.Println(key, val)
	state.innerState.Set(key, val)
}

func NewPrintSetState() *PrintSetState {
	return &PrintSetState{ograph.NewState()}
}

func TestCustomState(t *testing.T) {
	NewPrintSetState().Set("k", "v")
}
```

In the code, `PrintSetState` combines the default implementation and prints the additional parameters when calling the `Set` method, simulating the Debug scene.

## The type of key and the utility function.

You may have noticed that the type of `key` in several methods of `State` is `any`, which is the same as the design of the `context` package in `go`.

Specifically, considering that the nodes in the `pipeline` may come from different libraries, it is possible that two data points have no relationship, but their `key` is the same.

To avoid this situation, the type of the key is also involved in the comparison. If the type of the key is non-exported, then it is guaranteed that only nodes defined within the package can read the data set by the own node.

At the same time, in order to simplify the read-write of `State`, `ograph` also provides several utility functions:

{{< table >}}
| Functions                                                                                                              |
| ---------------------------------------------------------------------------------------------------------------------- |
| func LoadPrivateState[SK ~string, SV any](state ogcore.State, key string) SV                                           |
| func LoadState[SV any](state ogcore.State, key string) SV                                                              |
| func SavePrivateState[SK ~string](state ogcore.State, key string, val any, overwrite bool)                             |
| func SaveState(state ogcore.State, key string, val any, overwrite bool)                                                |
| func UpdatePrivateState[SK ~string, SV any](state ogcore.State, key string, updateFunc func(oldVal SV) (val SV)) error |
| func UpdateState[SV any](state ogcore.State, key string, updateFunc func(oldVal SV) (val SV)) error                    |
{{< /table >}}

These functions are essentially calling several methods of the `State` interface, the focus is on the automatic type conversion through generic parameters.