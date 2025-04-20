---
weight: 202
title: "事件"
description: "关于如何发送事件和监听事件"
icon: "article"
date: "2025-04-20T17:38:13+08:00"
lastmod: "2025-04-20T17:38:13+08:00"
draft: false
toc: true
---

事件一般用于代码解耦。在 OGraph 中，可以让节点发送事件，然后在 pipeline 上监听节点所发送的事件。

例如，节点可以发送一个代表发生了错误的事件，pipeline 捕获到这个事件后可以通过 webhook 上报。

这样节点只要专注于自己的业务逻辑，在 pipeline 层面也不用针对某个节点做错误处理。

## 示例

要让节点支持发送事件，那么首先需要让节点继承（内嵌）ograph.BaseEventNode，然后节点就可以调用 Emit 方法发送事件了。

```go
type TEventNode struct {
	ograph.BaseEventNode
}

func (node *TEventNode) Run(ctx context.Context, state ogcore.State) error {
	state.Set("msg", "hi, it is a test event.")
	node.Emit("test", state) // pass event info by state
	return nil
}
```

然后就是在 pipeline 中监听和处理事件了。

```go
	pipeline := ograph.NewPipeline()

	pipeline.Subscribe(func(event string, obj ogcore.State) bool {
		msg, _ := obj.Get("msg")
		fmt.Printf("get message from %s event: %v\n", event, msg)
		return true
	}, eventd.On("test"))

	pipeline.Register(ograph.NewElement("n").UseNode(&TEventNode{}))

	if err := pipeline.Run(context.Background(), nil); err != nil {
		fmt.Println(err)
	}
```

OGraph 使用了 github.com/symphony09/eventd 库来实现事件机制，eventd.On("test") 表示监听 test 事件，支持正则匹配事件名，详细用法可以参考 eventd 库的文档。

输出结果如下：

```
get message from test event: hi, it is a test event.
```

## 同步和异步

遵循 go 的哲学， OGraph 发送事件默认是同步的。也就是说 `node.Emit()` 会阻塞节点，如果你需要异步，那么就用 `go node.Emit()`。
