---
title: "Go 调度器(GMP)"
summary: "讲解 Go 运行时调度器的 G/M/P 三实体模型：本地队列与 work stealing、系统调用时的 P 让渡、基于信号的异步抢占、GOMAXPROCS 的容器陷阱与观测工具"
date: 2026-09-24
categories: [Programming, Go, Runtime]
tags: [go, concurrency]
---

## 概述

- 参考文档：
  - [The Go Blog: Go's work-stealing scheduler](https://go.dev/blog/waza-scheduler)
  - [Scalable Go Scheduler Design Doc 2012](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw)
  - 源码：[src/runtime/runtime2.go](https://github.com/golang/go/blob/master/src/runtime/runtime2.go)
- GMP 是 Go 运行时的调度模型：把用户级 goroutine(`G`)复用在一组 OS 线程(`M`)上，靠逻辑处理器(`P`)的持有关系把两者串起来
- 设计目标：让`go func()`的成本逼近一次函数调用，同时榨满多核，语言层的并发原语见[并发编程](/2025/12/25/go-concurrent/)
- 三者关系：

  ```mermaid
  flowchart LR
    subgraph M1["M (OS 线程)"]
      P1["P #1"] --> RQ1["本地队列 256"]
      RQ1 --> G1["G"]
    end
    subgraph M2["M (OS 线程)"]
      P2["P #2"] --> RQ2["本地队列"]
      RQ2 --> G2["G"]
    end
    GRQ["全局队列"] -.->|"每 61 次调度照顾一次"| P1 & P2
    RQ1 -.->|"work stealing 偷一半"| P2
  ```

## 快速开始

- 最直接的观察手段：`GODEBUG=schedtrace=1000 ./app`每秒打一行调度器快照
- 代码侧查询：`runtime.NumGoroutine()`看存量，`runtime.GOMAXPROCS(0)`只查不改
- 调度发生的起点就是最普通的语句：

  ```go
  go heavy()   // 新 G 进入调度体系，何时何地跑由运行时决定
  ```

## G、M 与 P

- G(`goroutine`)：
  - 对应结构体`runtime.g`，核心字段是可增长的栈与保存切换上下文的`gobuf`
  - 新 G 起始栈约`2KB`，按需成倍扩容，扩容时旧栈整体搬迁(连续栈)，所以 goroutine 比线程便宜一个数量级
  - 状态在`_Grunnable`、`_Grunning`、`_Gwaiting`、`_Gsyscall`、`_Gdead`之间流转，切换只换寄存器组**不陷入内核**
- M(`machine`)：
  - 真正的 OS 线程，对应`runtime.m`，执行 Go 代码前必须先绑定一个 P
  - 默认上限`10000`个，超限程序直接崩溃，`debug.SetMaxThreads`可改
  - `runtime.LockOSThread()`把 G 与 M 锁死，GUI 与部分 cgo 场景需要
- P(`processor`)：
  - 执行 G 的令牌，**数量等于`GOMAXPROCS`**，默认取机器逻辑核数
  - 每个 P 持有无锁的本地 G 队列(容量`256`)、`runnext`高优插槽与`mcache`等缓存
  - `GOMAXPROCS`限制的是同时执行 Go 代码的 M 数，**阻塞在系统调用里的 M 不占配额**

## 调度流程

- M 找 G 的顺序(`findRunnable`)：
  - 先看当前 P 的`runnext`，再扫本地队列
  - 每隔`61`次调度查一次全局队列，防止全局队列里的 G 饿死
  - 非阻塞地问一次 netpoller
  - 从别的 P 偷走其本地队列的一半(work stealing)
  - 还没有就阻塞等 netpoll，仍无活则解绑 P 并把 M 休眠
- 入队：`go`语句把新 G 放进当前 P 的本地队列，队列满时批量挪一半到全局队列
- 自旋 M：暂时没活干的 M 会先自旋观察一阵而不是立刻睡眠，**自旋数量被压在繁忙 P 的一半以内**，用少量空转换取新任务到达时的即时响应

## 系统调用与网络

- 阻塞系统调用(文件 IO 等)：
  - G 进入`_Gsyscall`，M 陷入内核，**M 与 P 解绑**，P 可被立刻交给其它 M 或新生的 M
  - 昂贵的线程阻塞被 P 隔离成局部事件，队列里的其它 G 不受影响
  - syscall 返回后原 M 尝试重新取 P，取不到就把 G 放入全局队列、自己休眠
- 网络 IO 走 netpoller(epoll 等后端)：
  - G 发起非阻塞读写，未就绪则挂起为`_Gwaiting`，**不占用任何线程**
  - 事件就绪后 netpoller 把 G 重新入队，这就是"阻塞式 API、非阻塞式成本"的实现

## 抢占

- Go 1.14 之前是协作式：靠函数序言的栈检查与隐式调度点，**一个无函数调用的死循环就能卡死 GC 与整个 P**
- Go 1.14 起加上基于信号的异步抢占：
  - 后台线程`sysmon`发现某 G 连续运行超过`10ms`
  - 向其所在 M 发送`SIGURG`，信号处理器在安全点把 G 切出
- 常见让出点：通道收发、锁竞争、`runtime.Gosched()`、GC 安全点、系统调用

## `GOMAXPROCS` 与容器

- 默认值取机器逻辑核数，**容器里它看不到 cgroup 的 CPU 配额**：
  - 8 核机器限`1`核配额，运行时仍按`8`个 P 满负荷发车，被节流(throttled)造成长尾延迟
  - 解法：环境变量`GOMAXPROCS`、`runtime.GOMAXPROCS(n)`，或引入`go.uber.org/automaxprocs`在启动时按 cgroup 自适应
- `GOMAXPROCS=1`时程序仍是并发的，只是同一时刻只有一个 G 在执行

## 观测工具

- `GODEBUG=schedtrace=1000`：一行快照，含各队列长度与线程数
- `GODEBUG=schedtrace=1000,scheddetail=1`：更细但噪音大，短时排查用
- `go tool trace`配合`runtime/trace`：可视化时间线，看 G 何时等锁、何时被抢占
- goroutine profile：`net/http/pprof`或`runtime/pprof`，G 数量只涨不降就是泄漏

## 相关阅读

- [并发编程](/2025/12/25/go-concurrent/)：channel、锁这些语言层原语，本文讲它们跑起来之后的事
- [Linux 进程管理命令](/2026/09/28/linux-process-cli/)：Go 的 M 在系统侧就是普通线程
- [操作系统的进程管理](/2025/04/03/os-process/)：调度理论侧
