---
title: Modern Objective-C
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - cocoa
  - Modern
  - OBJC
  - Objective-C
  - Style
---
## 前言
Objective-C已经出道了30多年，即使是iOS，也已经面世了5年之久，期间Objective-C和LLVM Clang都已经有了巨大的变化，这篇文章主要是描述一下Modern Objective-C Style。

在写这篇文章的时候，我本打算完整的描述一下我理解的现代Objective-C是如何，然后发现唐巧的blog上已经有一了一篇很好的博文：[Objective-C新特性](http://blog.devtang.com/blog/2012/08/05/use-modern-objective-c/)，大家可以先行阅读，我主要对他的博文进行补充。

## 使用ARC
我不想解释更多了，ARC虽说不是万能的，但在工程效率高于一切，速度就是王道的今天，ARC可以省下大量的时间。更何况，为什么总有人认为这些机械的内存管理，自己要做的比机器好呢？

想了解更多的ARC的信息，推荐Ray的arc教程

* http://www.raywenderlich.com/5677/beginning-arc-in-ios-5-part-1
* http://www.raywenderlich.com/5773/beginning-arc-in-ios-5-tutorial-part-2
## New Object Literals 和 Subscripting
参见唐巧的博文
## 关于instance variables和property和methods
我之前写了一篇[博文](http://gracelancy.com/?p=88)，引起了一些朋友的讨论。这里主要描述，在modern objective-c下，我们怎么写类里面的各种成员。

1. 你不再需要声明声明property的时候又声明instance variables，当你声明一个property的时候，编译器就会自动的帮你声明一个instance variable。
2. 你不再需要@synthesize，当你声明了@property的时候，编译器会自动帮你@synthesize，同时帮你指定好你的instance variable的名称为_var。
3. 即使你确实喜欢自己声明instance variables和自己@synthesize，那么你也不应该违反_var的命名形式。
4. 你不再需要Forward declaration，（只适用OBJC）
5. **把所有私有的东西从.h文件移动到.m文件**

强调一下，上面说的第五条，把所有私有的东西从.h文件移动到.m文件，可以说就是“old-fashioned” 和 modern OBJC的最大区别。具体有如下几点：

1. 你可以把需要私有的property从h文件移到m文件，这样外界将无法直接访问。
    * 记住IBOutlet也是可以移到m文件的。
2. 你可以把私有的instance variables从h文件移到m文件。这里可以移到两个地方：
    * .m文件的@interace
    * .m文件的@implementation
3. 你可以把类需要实现的protocol从h文件移到m文件，当你认为其不需要暴露给外部知道时。
4. 如之前所说，你不再需要前置声明，所以私有方法，可以直接从.h文件中删除。当然你也可以放在.m文件的@interface里。
    * 记住IBOutlet也可以直接连到.m文件中，不需要连到@interface里面做前置声明。
    
注意：有时候你会奇怪，为什么有的程序有instance variables，有些却没有。这里暂时还没有一个明确的定论，只是不同风格有不同的写法。有些人喜欢为所有的东西创建property，而有些人却不愿意为私有成员创建property。

就我个人而言，是习惯于为所有的东西创建property的，然后将私有的proerty移到m文件中。为什么使用property呢？这里有一个来自苹果工程师Paul（在stanford上课的那位）在课上的解释：

Why property？

Most importantly, it provides safety and subclassablility for instance variables. Also provides “value” for lazy instantiation, UI updating, consistency checking, etc.

## 使用Block
* 可以使用block来代替delegate
* 可以使用block来遍历容器。

Block通常意味着Do more with less code

## 关于property的权限，对外readonly对内readwrite的property
你可以在.h文件里设置一个权限readonly的property，并在.m文件设置一个权限为readwrite的同样的property，这时，你的property对外是只读的，对内却是可读写。

EDIT:补充关于最后这个的样例说明
在类的.h文件的interface下声明一个

```objective-c
@property (nonatomic, strong, readonly) NSString *testString;
```

再在.m文件的interface下声明一个

```objective-c
@property (nonatomic, strong, readwrite) NSString *testString;
```
    
此时在类内部可以对该变量进行修改，但在类外部修改会被编译器报错为readonly。



