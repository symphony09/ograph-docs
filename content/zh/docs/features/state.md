---
weight: 999
title: "State"
description: ""
icon: "article"
date: "2025-03-02T15:47:27+08:00"
lastmod: "2025-03-02T15:47:27+08:00"
draft: false
toc: true
---

在 ograph 中，节点间使用 `State` 共享数据。具体来说，`State` 是一个数据容器，在节点运行时作为参数传入，节点可以写入自己的数据，也可以从中读取其他节点写入的数据。

## 如何使用

`State` 是节点的运行参数之一，在运行期间可以调用相应方法进行读写操作，示例代码如下：

```go
func (printer *Printer) Run(ctx context.Context, state ogcore.State) error {
	// Get data from state by key.
	fmt.Println(state.Get("text"))
}
```

## 接口定义

为了适应不同场景的需要，`state` 设计为了接口。接口定义如下：

```go
type State interface {
	Get(key any) (any, bool)
	Set(key any, val any)
	Update(key any, updateFunc func(val any) any)
}
```

如代码所示，`State` 接口包含三个方法：

1. **Get**: 用于读取数据，参数为目标数据对应的键，第一个返回值为目标数据值，如果值不存在，那么第二个返回值应该为 `false`。
2. **Set**：用于写入数据，第一个参数为键，第二个参数为值。
3. **Update**：类似 `Set`，用于更新数据。第一个参数是键，第二个参数是一个函数，用于接收旧值返回新值。



## 默认 State

大部分情况下，你不需要自己去实现一个 `State`。你可以按以下代码获取默认的 `State` 来使用。

```go
// declare
state := ograph.NewState()

// init
state.Set("param_key", "param_val")

// pass it to pipeline
pipeline.Run(context.TODO(), state)
```

你也可以直接传递空值给 pipeline，这种情况下 pipeline 会自动创建一个默认的 `State` 给节点使用。

{{< alert context="info" text="默认的 State 在写入数据时会开启锁保护保障并发安全，请注意来避免阻塞。" />}}

## 自定义 State

就像前面所说的，一般情况下，你不需要自己去自定义实现一个 `State`。但是默认的实现不可能支持所有的业务场景，在以下情况下，你可能需要自定义 `State`。

1. 更高的高性能。默认的 `State` 使用读写锁来保障并发安全，这在一定程度上牺牲了性能，如果你用其他方法实现了并发安全，那么可以用无锁 `State` 替换默认实现。
2. 业务需求。比如**持久化功能**、**读写拦截功能**，**远程读写功能**(`redis`, `etcd`...)等。

在实现自定义 `State` 时，推荐使用组合的方式，比如：

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

代码中，`PrintSetState` 组合了默认实现，并在调用 `Set` 方法时额外打印参数，模拟了 Debug 的场景。

## Key 的类型和工具函数

你可能注意到了，在 `State` 的几个方法中，`key` 的类型都是 `any`。这和 `go` 中 `context` 包的设计是一样的。

具体来说，考虑到 `pipeline` 的节点可能来自不同的库，那么有可能出现两个数据毫无关系，但是 `key` 一样的情况。

为了避免这种情况，键的类型也参与对比。如果键的类型是非导出的，那么还可以保证只有包内定义节点才能读取到自己节点设置的数据。

同时为了简化 `State` 的读写，`ograph` 还提供了几个工具函数：

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

这些函数本质上还是调用 `State` 接口的几个方法，重点在于通过泛型参数做了自动类型转化。