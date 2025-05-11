---
weight: 202
title: "Events"
description: "How to send events and listen for events"
icon: "article"
date: "2025-04-20T17:38:13+08:00"
lastmod: "2025-04-20T17:38:13+08:00"
draft: false
toc: true
---

Events are generally used for code decoupling. In OGraph, nodes can send events, and pipelines can listen for events sent by nodes.

For example, a node can send an event representing an error occurrence. After the pipeline captures this event, it can report it via webhook.

This way, nodes only need to focus on their business logic, and the pipeline layer doesn't need to handle errors for specific nodes.

## Example

To enable a node to send events, first make the node inherit (embed) ograph.BaseEventNode. Then the node can call the Emit method to send events.

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

Then, listen and handle events in the pipeline.

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

OGraph uses the github.com/symphony09/eventd library to implement the event mechanism. eventd.On("test") indicates listening for the "test" event, supporting regular expression matching for event names. For detailed usage, please refer to the eventd library's documentation.

Output result:
```
get message from test event: hi, it is a test event.
```

## Synchronous and Asynchronous

Following Go's philosophy, OGraph sends events synchronously by default. That is, `node.Emit()` will block the node. If you need asynchronous processing, use `go node.Emit()`.
