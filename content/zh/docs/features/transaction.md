---
weight: 202
title: "事务"
description: "关于如何使用事件实现原子性"
icon: "article"
date: "2025-04-20T16:53:48+08:00"
lastmod: "2025-04-20T16:53:48+08:00"
draft: false
toc: true
---

## 示例

要让节点支持事务，首先要为节点实现 Commit 和 Rollback 方法

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

事务节点在注册时，和一般节点并没有什么不同。两者的区别在于，事务节点在 pipeline 执行结束的时候进行额外的提交逻辑（pipeline 运行成功时）或回滚逻辑（pipeline 运行报错时）。

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
上面代码的输出结果如下：

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

可以看到第一个pipeline运行成功，t1 和 t2 节点先是按顺序运行，然后按顺序提交。

第二个 pipeline t1 节点报错，所以对 t1 节点执行了回滚，而 t2 节点因为尚未运行，所以不会执行回滚。

## 多个事务

在上面的示例中可以看到，事务的生命周期是和 pipeline 绑定的。

pipeline 开始执行，就是开启了一个事务，然后一个或多个事务节点会被执行。当 pipeline 结束，事务也就到了最终的提交或回滚阶段。

同时，OGraph 支持包含子pipeline。因此，如果要在 pipeline 中把多个事务节点划分到不同事务中时，你可以创建不同的子 pipeline 嵌套在主 pipeline 中。此时，只要子pipeline 运行成功，那么它包含的事务节点就会被提交，而不用等主 pipeline 运行结束。

关于嵌套 pipeline 请参考 [子 Pipeline](../features/sub_pipeline.md)。