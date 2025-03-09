---
weight: 999
title: "Benchmark"
description: ""
icon: "Speed"
date: "2025-03-09T22:57:32+08:00"
lastmod: "2025-03-09T22:57:32+08:00"
draft: false
toc: true
---

**Performance Testing**

To better measure OGraph's performance, this test uses CGraph (a high-performance C++ scheduling framework) as a benchmark and compares it with go-taskflow (a Go-based scheduling framework).

Four scenarios were run under identical conditions to compare their differences.

### Test Code

For fairness, the OGraph test case adopts the same design as CGraph. The four scenarios are:

1. 32 nodes with no dependencies between them
2. 32 nodes connected serially (n1->n2->...->n32)
3. A simple DAG graph with six nodes
4. 64 nodes divided into eight layers, each layer containing eight nodes, and full connections between layers

OGraph Script: [github.com/symphony09/ograph/test/benchmark.sh](https://github.com/symphony09/ograph/blob/main/test/benchmark.sh)

OGraph Test Case：[github.com/symphony09/ograph/test/benchmark\_test.go](https://github.com/symphony09/ograph/blob/main/test/benchmark_test.go)

CGraph Script[github.com/ChunelFeng/CGraph/CGraph-run-performance-tests.sh](https://github.com/ChunelFeng/CGraph/blob/main/CGraph-run-performance-tests.sh)

CGraph Test Case：[github.com/ChunelFeng/CGraph/test/Performance](https://github.com/ChunelFeng/CGraph/tree/main/test/Performance)

go-taskflowTest Case：[github.com/noneback/go-taskflow/benchmark/benchmark_test.go](https://github.com/noneback/go-taskflow/blob/main/benchmark/benchmark_test.go)

## Test Result

### OGraph

    [pipeline 循环执行 100 万次]
    goos: linux
    goarch: amd64
    pkg: github.com/symphony09/ograph/test
    cpu: AMD Ryzen 5 5600G with Radeon Graphics         
    BenchmarkConcurrent_32-8         1000000              4308 ns/op            1592 B/op         16 allocs/op
    BenchmarkSerial_32-8             1000000               281.7 ns/op           120 B/op          4 allocs/op
    BenchmarkComplex_6-8             1000000              2762 ns/op            1048 B/op         21 allocs/op
    BenchmarkConnect_8x8-8           1000000              8333 ns/op            2553 B/op         16 allocs/op
    PASS
    ok      github.com/symphony09/ograph/test       15.692s

### CGraph

Unlike OGraph, where each test case is executed 1 million times, CGraph's four test cases are executed 500k, 1 million, 1 million, and 100k times respectively.

    **** Running test-performance-01 ****
    [2024-12-23 21:07:45.992] [test_performance_01]: time counter is : [4102.16] ms
     
    **** Running test-performance-02 ****
    [2024-12-23 21:07:46.567] [test_performance_02]: time counter is : [572.51] ms
     
    **** Running test-performance-03 ****
    [2024-12-23 21:07:50.611] [test_performance_03]: time counter is : [4042.16] ms
     
    **** Running test-performance-04 ****
    [2024-12-23 21:07:51.958] [test_performance_04]: time counter is : [1345.02] ms
     
     congratulations, automatic run CGraph performance test finish...


### go-taskflow
    goos: linux
    goarch: amd64
    pkg: github.com/noneback/go-taskflow/benchmark
    cpu: AMD Ryzen 5 5600G with Radeon Graphics
    BenchmarkC32-8             39032             30992 ns/op            7465 B/op        227 allocs/op
    BenchmarkS32-8             15487             77402 ns/op            6908 B/op        255 allocs/op
    BenchmarkC6-8             108658             10948 ns/op            1297 B/op         46 allocs/op
    BenchmarkC8x8-8            14022             85813 ns/op           16957 B/op        503 allocs/op
    PASS
    ok      github.com/noneback/go-taskflow/benchmark       6.899s

## Test Result Analysis

The number of pipeline execution times for the 4 test cases of CGraph are 500k, 1000k, 1000k, and 100k.
The time taken per execution is calculated to be 8204 ns/op, 572 ns/op, 4042 ns/op, and 13450 ns/op respectively.

|                                            | CGraph (Baseline) | OGraph (This Project) | go-taskflow |
| :----------------------------------------- | :---------------: | :-------------------: | :---------: |
| Scenario One (No Connection for 32 Nodes)  |    8204 ns/op     | 4308 ns/op（+90.4%）  | 30992 ns/op |
| Scenario Two (Serially Connected 32 Nodes) |     572 ns/op     | 281.7 ns/op（+103%）  | 77402 ns/op |
| Scenario Three (Simple DAG with Six Nodes) |    4042 ns/op     | 2762 ns/op（+46.3%）  | 10948 ns/op |
| Scenario Four (8x8 Full Connection)        |    13450 ns/op    | 8333 ns/op（+61.4%）  | 85813 ns/op |

From the test results of the four scenarios, OGraph’s performance is comparable to that of CGraph and even surpasses it in many cases.

During testing, OGraph's CPU usage was also noticeably lower than CGraph's.

## Concurrency Performance

OGraph pipeline supports concurrent execution. The concurrency test results for the same scenarios are as follows:

|                                            | OGraph (Pipeline Loop Execution) | OGraph (Parallel Pipeline Execution) |
| :----------------------------------------- | :------------------------------: | :----------------------------------: |
| Scenario One (No Connection for 32 Nodes)  |            4308 ns/op            |         3138 ns/op（+37.2%）         |
| Scenario Two (Serially Connected 32 Nodes) |           281.7 ns/op            |         57.25 ns/op（+392%）         |
| Scenario Three (Simple DAG with Six Nodes) |            2762 ns/op            |        747.8 ns/op（+269.3%）        |
| Scenario Four (8x8 Full Connection)        |            8333 ns/op            |         5136 ns/op（+62.2%）         |

Test Script: [ograph/test/benchmark\_parallel.sh at main · symphony09/ograph](https://github.com/symphony09/ograph/blob/main/test/benchmark_parallel.sh)

## Appendix

### Test Hardware

**CPU:** AMD 5600G - 6 cores, 12 threads

**Memory:** DDR4 32 GB

### Test Software

**System:** Linux 6.11 - Fedora Workstation 41

**OGraph:** v0.6.1 - Go 1.23.4

**CGraph:** v2.6.2 - GCC14.2.1