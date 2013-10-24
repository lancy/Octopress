---
title: OS X/iOS 并发编程小结
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - GCD
  - iOS
  - Operation Queue
  - 并发编程
---

## 简洁
这里不讨论传统的多线程编程，而讨论OS X/iOS特有的异步技术，GCD和Operation Queue。

* **Grand Central Dispatch (GCD)：** C语言的系统管理线程，不需要编写线程代码，只需要定义需要执行的任务，然后以block的形式添加到适当的dispatch queue中，系统就会负责线程的创建和任务调度。
* **Operation Queue：** Cocoa的系统管理线程，与GCD类似。


## Operation Queues
基于cocoa的应用通常会使用Operation Queues

### Operation Objects
NSOperation是抽象基类，需要实现相应子类。不过Cocoa提供了两个具体的子类NSInvocationOperation和NSBlockOperation。其中NSInvocationOperation以@selector来创建operation object；NSBlockOperation以block来创建operation object.(这里暂且不讨论自己实现NSOperation).

所有的operation objects都支持这些特性：

* 依赖关系，可以阻塞某个operation，直到他所依赖的所有operation都已经完成
* 可以设置completion block
* 可以通过KVO来监控operation的状态
* 可以设置operation的优先级
* 可以取消

创建operation object之后，加入到适当的operation queue即会立刻开始执行。

### Operaton queue
* 可以设置并发执行的operation 数量，设为1，即为串行队列
* 可以暂时挂起，继续，等待直到完成


## Dispatch Queues
GCD使用block来创建任务，切任务总是以添加的顺序开始顺序执行。有串行队列，也有并发队列，还有主线程队列。

注：GCD相关技术还有，Dispatch group：监控一组block对象完成；Dispatch semaphore：类似于传统的信号量；Dispatch source：在特定类型的系统事件发生时产生通知。

### 管理和创建Dispatch Queue
* dispatch_get_global_queue 获得全局共享并发队列
* dispatch_queue_create 创建串行队列
* 可以通过dispatch_set_context和dispatch_get_context来管理自定义的上下文信息，finalizer可以销毁上下文。
* dispatch_async 异步添加任务到queue，dispatch_sync同步添加（尽量少用）
* 据说可以添加completed block，但是实际上就是再添加一个block，（感觉略无力，不敢用）
* dispatch_apply可以执行循环迭代

### Dispatch Semaphore
1. dispatch_semaphore_create创建信号量，指定可用资源数
2. dispatch_semaphore_wait等待可用资源数
3. dispatch_semaphore_signal释放信号量

### Dispatch group
1. dispatch_group_create 创建group
2. dispatch_group_async 添加到group
3. dispatch_group_wait 阻塞线程直到group完成

### Dispatch source
略

----
备注：queue 不是替代线程的万能药!queue 提供的异步编程模型适合 于延迟无关紧要的场合。虽然 queue 提供配置任务执行优先级的方法, 但更高的优先级也不能确保任务一定能在特定时间得到执行。因此线程 仍然是实现最小延迟的适当选择,例如音频和视频 playback 等场合。

（上面这一条让我吐死。。看了几十页的文档，写了n多代码。。发现real time是不适用的。。然后开始用多线程。。过两天在补一个多线程编程小结。。）