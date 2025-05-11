---
weight: 202
title: "Transactions"
description: "About how to achieve atomicity using events"
icon: "article"
date: "2025-04-20T16:53:48+08:00"
lastmod: "2025-04-20T16:53:48+08:00"
draft: false
toc: true
---

## Example

Before a node can support transactions, you need to implement the Commit and Rollback methods for the node

```go
func (tx *TxNode) Run(ctx context.Context, state ogcore.State) error {
	fmt.Println("run tx", tx.Name())
	return nil
}

func (tx *TxNode) Commit() {
	fmt.Println("commit tx", tx.Name())
}

func (tx *TxNode) Rollback() {
	fmt.Println("rollback tx", tx.Name())
}
```

Transaction nodes are registered in the same way as regular nodes. The difference between transaction nodes and regular nodes is that transaction nodes perform additional commit logic (when the pipeline runs successfully) or rollback logic (when the pipeline runs with errors) at the end of the pipeline execution.

```go
	pipeline := ograph.NewPipeline()
	exceptErr := errors.New("except error")

	n1 := ograph.NewElement("n1").UseFn(func() error {
		return nil
	})
	n2 := ograph.NewElement("n2").UseFn(func() error {
		fmt.Println("an error occurred")
		return exceptErr
	})
	t1 := ograph.NewElement("t1").UseNode(&TxNode{})
	t2 := ograph.NewElement("t2").UseNode(&TxNode{})

	pipeline.Register(n1, ograph.Branch(t1, t2))

	fmt.Println("[pipeline without error]")
	if err := pipeline.Run(context.TODO(), nil); err != nil {
		fmt.Println(err)
	}

	pipeline = ograph.NewPipeline()
	pipeline.Register(n1, ograph.Branch(t1, n2, t2))

	fmt.Println("[pipeline with error]")
	if err := pipeline.Run(context.TODO(), nil); errors.Unwrap(err).Error() != "except error" {
		fmt.Println(err)
	}
```
The output of the above code is as follows:

```
[pipeline without error]
run tx t1
run tx t2
commit tx t1
commit tx t2
[pipeline with error]
run tx t1
an error occurred
rollback tx t1
```

As you can see, the first pipeline runs successfully, and the t1 and t2 nodes are executed in sequence, then committed in sequence.

In the second pipeline, the t1 node returns an error, so the rollback of the t1 node is executed, while the t2 node has not been executed yet, so no rollback is performed.

## Multiple Transactions

As shown in the example above, the lifecycle of a transaction is bound to the pipeline.

When the pipeline starts executing, a transaction is initiated. Then one or more transaction nodes will be executed. When the pipeline ends, the transaction will reach the final commit or rollback stage.

At the same time, OGraph supports nested sub-pipelines. Therefore, if you want to group multiple transaction nodes into different transactions within a pipeline, you can create different sub-pipelines nested within the main pipeline. In this case, as long as the sub-pipeline runs successfully, the transaction nodes it contains will be committed, without waiting for the main pipeline to finish running.

For more information about nested pipelines, please refer to [Sub Pipeline](../features/sub_pipeline.md).
