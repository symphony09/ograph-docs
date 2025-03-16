---
weight: 301
title: "Built-in Nodes"
description: "built-in nodes reference"
icon: "article"
date: "2025-03-16T22:55:55+08:00"
lastmod: "2025-03-16T22:55:55+08:00"
draft: false
toc: true
---

## Cmd Node

> General-purpose nodes for executing command-line commands can be cross-platform used.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
	p := ograph.NewPipeline()

	e := ograph.NewElement("cmd").UseFactory(ogimpl.CMD).
		Params("Cmd", []string{"go", "version"})

    // register and run the node which exec go version
	p.Register(e).Run(context.TODO(), nil)
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name | Required | Meaning           | Type     | Example          |
| :--- | :------- | :---------------- | -------- | :--------------- |
| Cmd  | ✔        | cmd to exec       | []string | ["go","version"] |
| Env  | ✗        | exec env          | []string | ["key=value"]    |
| Dir  | ✗        | working directory | string   | "/root"          |

Cmd parameter format: ["commandName/path", "parameter1", "parameter2", ...]

{{% /tab %}}

{{% tab tabName="Tips" %}}
Use the OGRAPH_ALLOW_CMD_LIST environment variable to limit executable commands

**Linux**

```bash 
export OGRAPH_ALLOW_CMD_LIST=ls,cat
```
{{% /tab %}}
{{< /tabs >}}

## Http Req Node

> Node for initiating HTTP requests, used for scenarios such as webhooks.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
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

{{% tab tabName="Parameter" %}}
| Name        | Required | Meaning       | Type   | Example                         |
| :---------- | :------- | :------------ | ------ | :------------------------------ |
| Method      | ✔        | http method   | string | POST                            |
| Url         | ✔        | http url      | string | http://localhost:8080/ping      |
| ContentType | ✗        | content-type  | string | application/json                |
| Body        | ✗        | request body  | string | hello                           |
| BodyTpl     | ✗        | body template | string | hello, i am {{GetState "name"}} |

{{% /tab %}}

{{% tab tabName="Tips" %}}
Currently supports only GET and POST request methods.

When rendering the request body with a template, the GetState function can be used to extract the execution state values. The state value must be publicly accessible (i.e., the key needs to be of string type, not a private type).
{{% /tab %}}
{{< /tabs >}}

## Cmd Node

> General-purpose nodes for executing command-line commands can be cross-platform used.

{{< tabs tabTotal="3">}}
{{% tab tabName="Basic Usage" %}}
```go
	p := ograph.NewPipeline()

	a := ograph.NewElement("Assert").Apply(ogimpl.AssertOp("i==1"))

	p.Register(a)

	s := ograph.NewState()
	s.Set("i", 1) // if set with other value, the assertion fails, then the pipeline will also fail

	p.Run(context.TODO(), s)
```
{{% /tab %}}

{{% tab tabName="Parameter" %}}
| Name       | Required | Meaning     | Type   | Example |
| :--------- | :------- | :---------- | ------ | :------ |
| AssertExpr | ✔        | Assert Expr | string | "i==1"  |

{{% /tab %}}

{{% tab tabName="Tips" %}}
The state value must be publicly accessible (i.e., the key needs to be of string type, not a private type).
{{% /tab %}}
{{< /tabs >}}