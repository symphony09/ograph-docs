---
weight: 301
title: "内置普通节点"
description: "内置节点的详细使用文档"
icon: "article"
date: "2025-03-16T22:28:43+08:00"
lastmod: "2025-03-16T22:28:43+08:00"
draft: false
toc: true
---

## 命令行节点

> 用于执行命令行的普通节点，可以跨平台使用

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
```go
	p := ograph.NewPipeline()

	e := ograph.NewElement("cmd").UseFactory(ogimpl.CMD).
		Params("Cmd", []string{"go", "version"})

    // register and run the node which exec go version
	p.Register(e).Run(context.TODO(), nil)
```
{{% /tab %}}

{{% tab tabName="参数说明" %}}
| 参数名 | 必需 | 含义              | 类型     | 示例             |
| :----- | :--- | :---------------- | -------- | :--------------- |
| Cmd    | ✔    | cmd to exec       | []string | ["go","version"] |
| Env    | ✗    | exec env          | []string | ["key=value"]    |
| Dir    | ✗    | working directory | string   | "/root"          |

Cmd 参数格式：["命令名/路径", "参数1", "参数2", ...]
{{% /tab %}}

{{% tab tabName="小贴士" %}}
使用 OGRAPH_ALLOW_CMD_LIST 环境变量限制可执行命令

**Linux**

```bash 
export OGRAPH_ALLOW_CMD_LIST=ls,cat
```
{{% /tab %}}
{{< /tabs >}}


## HTTP请求节点

> 用于发起 http 请求的节点，用于 webhook 等场景

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
```go
	state := ograph.NewState()
	state.Set("time", time.Now())

	p := ograph.NewPipeline()

	e := ograph.NewElement("req").UseFactory(ogimpl.HttpReq).
		Params("Method", "POST").
		Params("Url", "http://localhost:8080/ping").
		// Render body as:
		// Send-Time: 2024-08-03 10:10:20.0873372 +0800 CST m=+0.002574001
		Params("BodyTpl", `Send-Time: {{GetState "time"}}`)

	p.Register(e).Run(context.Background(), state)
```
{{% /tab %}}

{{% tab tabName="参数说明" %}}
| 参数名      | 必需 | 含义          | 类型   | 示例                            |
| :---------- | :--- | :------------ | ------ | :------------------------------ |
| Method      | ✔    | http method   | string | POST                            |
| Url         | ✔    | http url      | string | http://localhost:8080/ping      |
| ContentType | ✗    | content-type  | string | application/json                |
| Body        | ✗    | request body  | string | hello                           |
| BodyTpl     | ✗    | body template | string | hello, i am {{GetState "name"}} |

{{% /tab %}}

{{% tab tabName="小贴士" %}}
目前仅支持 GET, POST 两种请求方法。

使用模板渲染请求体时，可以使用 GetState 函数提取执行状态值。要求这个状态值是可以公开访问的（即 key 需要为 string 类型，而非私有类型）。

{{% /tab %}}
{{< /tabs >}}


## 断言节点

> 断言 State 中的值，可以及时发现错误并退出 pipeline

{{< tabs tabTotal="3">}}
{{% tab tabName="基本使用方式" %}}
```go
	p := ograph.NewPipeline()

	a := ograph.NewElement("Assert").Apply(ogimpl.AssertOp("i==1"))

	p.Register(a)

	s := ograph.NewState()
	s.Set("i", 1) // if set with other value, the assertion fails, then the pipeline will also fail

	p.Run(context.TODO(), s)
```
{{% /tab %}}

{{% tab tabName="参数说明" %}}
| 参数名     | 必需 | 含义       | 类型   | 示例   |
| :--------- | :--- | :--------- | ------ | :----- |
| AssertExpr | ✔    | 断言表达式 | string | "i==1" |

{{% /tab %}}

{{% tab tabName="小贴士" %}}
要求状态值是可以公开访问的（即 key 需要为 string 类型，而非私有类型）。
{{% /tab %}}
{{< /tabs >}}
