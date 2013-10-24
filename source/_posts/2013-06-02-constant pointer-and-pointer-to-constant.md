---
title: OBJC中声明字符串常量的一个常见错误（常量指针和指针常量）
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - cocoa
  - const
  - iOS
  - OBJC
  - pointer
  - string
---


我们知道，NSNotification是Cocoa中观察模式最易用的实现方法，比起直接使用KVO（Key-Value Observing）他更加容易实现也更好理解。一个样例：

#### Poster.h

```objective-c
// Define a string constant for the notification
extern NSString * const PosterDidSomethingNotification;
```
#### Poster.m

```objective-c
NSString * const PosterDidSomethingNotification = @”PosterDidSomethingNotification”;
...
// Include the poster as the object in the notification
[[NSNotificationCenter defaultCenter]
postNotificationName:PosterDidSomethingNotification
              object:self];
```
#### Observer.m

```objective-c
// Import Poster.h to get the string constant
#import “Poster.h”
...
// Register to receive a notification
[[NSNotificationCenter defaultCenter] addObserver:self 
                                         selector:@selector(posterDidSomething:) 
                                             name:PosterDidSomethingNotification 
                                           object:nil];
...
- (void) posterDidSomething:(NSNotification *)note {
    // Handle the notification here
}
- (void)dealloc {
    // Always remove your observations
    [[NSNotificationCenter defaultCenter]
    removeObserver:self];
    [super dealloc];
}
```
    

注意到，在使用Notifikation的时候，会需要声明字符串常量，作为notification的name。这时，const的位置就比较重要，很容易让不了解的人犯错误：
    
错误的写法（常量指针）：
```objective-c
extern const NSString * RNFooDidCompleteNotification;
```
正确的写法（指针常量）：

```objective-c
extern NSString * const RNFooDidCompleteNotification;
```
    

这里涉及到常量指针和指针常量的概念，简单的来说：

+ 常量指针：就是指向常量的指针，关键字 const 出现在 * 左边，表示指针所指向的地址的内容是不可修改的，但指针自身可变。
+ 指针常量：指针自身是一个常量，关键字 const 出现在 * 右边，表示指针自身不可变，但其指向的地址的内容是可以被修改的。
    
在此例中：我们知道，NSString永远是immutable的，所以NSString * const 是有效的，而const NSString * 则是无效的。而使用错误的写法，则无法阻止修改该指针指向的地址，使得本应该是常量的值能被修改，造成了隐患。这是需要注意的一个常见错误。