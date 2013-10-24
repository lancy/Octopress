---
title: OS X/iOS 多线程编程小结
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - iOS
  - thread
---

## 前言
我来填坑了。欠下的多线程，趁着假期无聊，赶紧码了。

好吧，如我的这篇博文《[OS X/iOS 并发编程小结](http://gracelancy.com/?p=116)》最后一段所说，虽然queue那样的并发编程非常方便，但在音频、视频这些需求最小延时的情况下，queue并发编程即使有优先级也并不能保证任务在特定的时间得到执行。故而，这种情况下还是需要我们直接操作线程。

注：除开这种情况外，我建议（苹果也建议），各位还是乖乖用queue并发编程就好了。

## 什么是多线程
简而言之，主要有三个术语：线程、进程、任务。
这里不想给大家上操作系统课啦，大家不明白的google一下或者回去翻翻书就好。

## APPLE的多线程技术
### APPLE关于多线程的整个架构
![image](http://lianxu.me/images/post/threadlayout.jpg)
（注：NSOperationQueue是在GCD上面的）

ok，我一个一个介绍：

1. 多线程底层实现的机制是Mach的线程。坦白的说，mach我压根不懂。苹果也说了，你几乎不会用到。不过如果你真的有兴趣研究，可以看这里：[Kernel Programming Guide](https://developer.apple.com/library/mac/#documentation/Darwin/Conceptual/KernelProgramming/Mach/Mach.html) 里面的关于Mach的那部分。
2. pthread(POSIX threads)。这是传统的多线程，C-based interface，非常灵活。如果你写的不是Cocoa应用，这是实现多线程的最好选择。
3. NSThread，Cocoa的线程实现。当你在写Cocoa应用而又需要进行直接的线程操作的时候，我认为NSThread是比pthread更好的选择。虽然你依然可以使用pthread在Cocoa程序上，但是会有一些在Cocoa程序需要遵循的规则。我会在后面说明。
4. GCD和NSOperationQueue略

### 同步工具
锁(locks)，条件(condition)，原子操作(atomic)

### 线程间通信
1. Direct messaging: Cocoa应用可以直接perform某个selector在指定线程
2. 全局变量，共享内存和对象。
3. Conditions：一种特殊类型的锁
4. Run loop sources：简单的来说，run loop是用来在线程上管理时间异步到达的工具。run loop能为线程监听一个或多个事件源(event sources)。run loop能把线程置于休眠状态，而当事件到达时，系统能唤醒线程并把事件分发给run loop，而后run loop能将事件分发给特定的handler。
5. Ports and sockets：也使用run loop，不同之处在于可以进行多进程通信
6. Message queues：历史遗物，一种多进程通信的玩意，才用FIFO的信息队列，但是有效率问题，
7. Cocoa distributed objects：好高级的技术，可以在call不同cocoa应用的object，甚至跨越网络的不同计算机上的cocoa应用。

（老实说，后面几种我原先压根就没见过，为了弄懂他们是啥，看了好几篇文档。总结到这里，我的压力也感觉越来越大。觉得自己图样图拿衣服。。写了点多线程代码，就想总结OSX\iOS多线程编程。这才发现里面内容之多，细节之细令人发指。远不是整理一个并发编程能比的。我水平恐怕也就能堪堪掌握个大概框架，就操作系统课上的那点东西，跟生产环节下的多线程相比真是弱爆了。。）

所以从这里开始，后面我打算走实用主义，直接小结一下如何写多线程程序好了。

## 线程管理
### 使用NSThread创建线程
* 类方法detachNewThreadSelector:toTarget:withObject:
* 创建一个NSThread对象initWithTarget:selector:object:，并调用start方法

需要注意的是，这两种方法创建的线程，是分离（detach）的线程。detached的意思即，当线程退出的时候，系统会自动回收线程资源。当线程运行的时候，可以使用performSelector:onThread:withObject:waitUntilDone:modes来进行线程通信。其中modes用来指定run loop

注意：在大部分情况下，脱离线程更适用，因为它允许系统在线程完成的时候自动回收。如果你想创建可连接的线程（Joinable thread），唯一的办法是使用pthread。

### 使用pthread创建线程
[参看pthread手册](http://developer.apple.com/library/ios/#documentation/System/Conceptual/ManPages_iPhoneOS/man3/pthread.3.html#//apple_ref/doc/man/3/pthread)

需要注意的几点：

1. 在cocoa程序上仅使用pthread，你需要先使用NSThread生成一个线程之后立即退出。就能保证Cocoa框架切换到多线程模式，来启用一些锁或者其他的同步技术来保证Cocoa框架代码的正确执行。如果你不这么做，当涉及到Cocoa框架的操作时有可能会照成不可预料的后果。
2. pthread和Cocoa的锁是完全可以混用的，（我就这么做过，因为pthread的锁更灵活），但是对于指定的一个锁，必须使用同类型的接口来操作，即你不能同时用pthread和NSLock同时操作一个锁。

### 使用NSObject创建线程
performSelectorInBackground:withObject:也会生成一个脱离线程。你还可以在该线程里面使用performSelectorOnMainThread:withObject:waitUntilDone:modes:

### 线程的优先级
创建的线程的优先级和所处在的线程是相同的，优先级高的线程比优先级低的线程能获得更多运行机会，但并不能保证线程的具体执行时间和顺序。内核的调度算法会决定该运行哪个线程。

NSThread 的 setThreadPriority:以及pthread_setschedparam 可以改变线程的优先级。

注意：一般来说，保持默认优先级是一个不错的选择。更改某些线程的优先级，会增加某些较低优先级的线程的饥饿。若高优先级和低优先级的线程又有交互的需求，那么低优先级就有可能因为得到运行机会的难度而阻塞其他线程，造成线程瓶颈。


### 关于自动释放池 （autorelease pool）
autorelease pool用于自动释放pool里面捕获的autorelease对象。

理论上来说，每一个线程都应该有一个autorelease pool，主
线程在main.m里面XCode会为你创建一个。但当你创建一个线程的时候，你第一件应该做的事，就是创建一个autorelease pool。

非ARC之前，你使用NSAutoreleasePool，之后你可以使用@autoreleasepool。

注意：在ARC环境下你仍然不能忽略autorelease pool，因为ARC仍然使用autorelease来进行release操作，且他并不会为你自动的创建autorelease pool。BTW，除了在线程里面你会需要autorelease pool之外，在次数巨大或者多重的for-loop里面你也会使用autorelease pool，来避免一次alloc和release数量巨大的对象。


### 中断线程

（未完待续。。。写不动了。。）


## Run Loop 详解

## 线程同步 详解

## Cocoa的线程安全


