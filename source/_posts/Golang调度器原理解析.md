---
title: Golang调度器原理解析
date: 2019-12-25 15:28:40
tags: 高并发
categories: 编程语言
description: 调度器 (Scheduler) 使得多个Goroutine可以在多个内核线程 (thread) 上运行。大部分Goroutine的切换没有OS线程切换时的开销，使得整体运行效率比OS线程调度效率高很多。
cover: /images/scheduler_concepts.png
---

## 前置知识
### 什么是Goroutine
Goroutine可以看作是轻量级的抽象thread。在编写Go代码的时候，我们会对Goroutine进行操作而不是针对thread。对于操作系统而言，thread是最小的调度单位。所以可以认为goroutine是用户层面的线程抽象。

### Goroutine和thread的区别
Goroutine与thread之间的区别<sup>[1]</sup>主要可以从三个方面出发，如下表所示：

 区别 | Goroutine  | Thread
------------- | ------------- | -------------
内存占用 | 占用2KB栈内存，根据需要在运行中扩容  | 占用1MB左右内存，同时创建guard page与其他线程隔离
创建和销毁 |  Go runtime负责管理，用户级资源，创建销毁消耗非常小 | 内核级，消耗很大
切换成本 | 只需要保存三个寄存器 (register)，一般200ns左右 | 需要保存各种寄存器，消耗1000-1500ns

### Scheduler
关于调度器有三种基本模型：
1. N:1。也就是多个用户线程对应一个系统线程。优势在于上下文切换快，缺点在于难以发挥多核处理器的优势；
2. 1:1。也就是一个用户线程对应一个系统线程。牺牲上下文切换成本，充分利用多核处理器的优势；
3. M:N。理论上能在上下文切换和多核处理器之间找到平衡。

Thread与Goroutine之间是一个M:N的关系。Go程序启动的时候，runtime会创建M个thread，之后创建的N个Goroutine会依附于这M个thread上执行。总的说来，Go runtime维护所有的Goroutine，通过scheduler进行调度。Goroutine与thread相互独立，但是Goroutine依托thread进行执行。

同一时刻一个thread上只能有一个Goroutine被执行。这时候什么thread上执行哪一个Goroutine，如何进行上下文的切换需要有一个中间人Scheduler做调度。Scheduler的调度也是Go程序高效执行的关键之一。

## 调度器（Scheduler）
### 调度模型MPG
Golang的scheduler主要由三个部分组成：

1. M（Machine）代表OS thread；
2. P（Processer）代表调度器（context for scheduling），通常P的数量等于CPU核数（**GOMAXPROCS**）；
3. G （Goroutine）代表Goroutine。

他们的具体关系如下图所示<sup>[2]</sup>:

![MPG Model](/images/mpg_scheduler.jpg "MPG Model")

上图揭示了几个要点<sup>[3]</sup>：

1. G需要绑定在M上才能运行；
2. M需要绑定在P上才能运行；
3. 程序中多个M并不会同时处于运行状态，最多只有 *GOMAXPROCS* 个M在运行。

在Go 1.1之前并没有P的存在。调度是由G与M共同完成。Global维护一个runqueue，当M需要G的时候便从runqueue中获取。这时候需要一个全局所来保护调度对象。很明显，全局锁的存在严重影响了Goroutine的并发性能。Go 1.1之后Dmitry Vyukov<sup>[4]</sup>设计了Processor对原先的Go scheduling进行了改进，使得每一个M上绑定一个P，P会维护一个runnable状态的G队列（Local Runnable Queue, LRQ），解决了原先全局锁的问题。

### 调度算法（Work-stealing）
通过引入P，实现的work-stealing调度算法如下：
1. 每一个P维护一个可运行队列LRQ；
2. 当一个G生成时将其放入一个P的LRQ中；
3. 当一个G执行结束时P会从队列中把G取出，如果P队列为空则会随机选择另外一个P偷取他一半的G；

Work-stealing本质上来讲是一个负载均衡的算法。除了LRQ之外runttime也会维护一个GRQ（Global Runnable Queue）存放没有被分配具体P的G。

### 同步系统调度（Blocking System Call）
之所以P存在，是当有M被sysCall block的时候，我们能够把整个P交给另外一个M继续执行。当sysCall执行完毕后M会偷取其他的P，如果无法找到合适的P，M会进入线程池休眠。

![Blocking System Call](/images/syscall.jpg "Blocking System Call")

### 异步系统调度（Asyn System Call）
如果是异步调用，M不会被阻塞。如图所示，G1的异步请求会被Network Poll接手，M此时会继续执行G2，当G1异步请求完成，会自动放回P的LRQ中，整个过程如下图所示<sup>[5]</sup>。

![Async System Call](/images/async_syscall.png "Async System Call")

### 抢占式调度（Takeover）
Goroutine的执行是可以被抢占<sup>[3]</sup>的。简单来说，如果一个Goroutine占用CPU时间过长（> 10ms），P长时间没有进行调度，runtime会将其抢占，把CPU时间交给其他Goroutine。

具体来看，runtime启动时，程序会创建一个系统线程，运行`sysmon()`函数，负责监控所有Goroutine的状态同时判断是否需要进行垃圾回收。如果G执行时间过长，`sysmon()`会对这个G进行标记，下一次函数调用的时候会自动失败，并且和对应的M解除绑定关系并移送全局可执行队列GRQ中，然后为P设置新的可执行G。

## 总结
Go并发效率如此之高，我们可以做一个简单的总结：

1. Goroutine相较线程来看更为轻量，创建、销毁以及上下文切换开销小很多；
2. Scheduler实现了M:N的调度模式（并行能力由GOMAXPROCS决定，也就是多少个Processor），兼顾N:1，1:1调度模型优点，整体运行效率比线程调度高很多。


## Reference
* [1] <https://blog.nindalf.com/posts/how-goroutines-work/> "How Goroutines Work"
* [2] <https://morsmachine.dk/go-scheduler> "The Go scheduler"
* [3] <https://cloud.tencent.com/developer/article/1165575> "Go runtime scheduler"
* [4] <https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit> "Scalable Go Scheduler Design Doc"
* [5] <https://qcrao.com/2019/09/02/dive-into-go-scheduler/> "深度解密Go语言之scheduler"