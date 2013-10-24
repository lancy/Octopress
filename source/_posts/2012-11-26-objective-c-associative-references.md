---
title: Objective-C Associative References(关联引用)
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - Associative
  - iOS
  - OBJC
  
---

## About 
在研究Objc的运行时特性的时候，发现了一个有意思的东东，Associative Reference关联引用。使用关联引用，能够模拟添加一个对象实例到一个已有的类中，能够添加存储到一个对象中而不需要改变类的定义。这个技术在你不能访问源码的时候有用，或者你只是觉得动态的增加关联很好玩。
## 创建关联
可以使用 objc_setAssociatedObject 来创建关联引用。

```objective-c
    static char overviewKey;
    NSArray *array = @[@"One", @"Two", @"Three"];
    NSString *overview = @"First three numbers";
    objc_setAssociatedObject (
                              array,
                              &overviewKey,
                              overview,
                              OBJC_ASSOCIATION_RETAIN
                              );
```

* key是一个 void 指针，对于每个关联，key必须唯一，通常可以使用一个 static variable.
* police 用来指定，关联的对象是 assigned, retained, copied, 还有是否是atomically.
    * **OBJC_ASSOCIATION_ASSIGN**
Specifies a weak reference to the associated object.
    * **OBJC_ASSOCIATION_RETAIN_NONATOMIC**
Specifies a strong reference to the associated object, and that the association is not made atomically.
    * **OBJC_ASSOCIATION_COPY_NONATOMIC**
Specifies that the associated object is copied, and that the association is not made atomically.
    * **OBJC_ASSOCIATION_RETAIN**
Specifies a strong reference to the associated object, and that the association is made atomically.
    * **OBJC_ASSOCIATION_COPY**
Specifies that the associated object is copied, and that the association is made atomically.

## 取回关联对象

```objective-c
NSString *associatedObject = (NSString *)objc_getAssociatedObject(array, &overviewKey);
```

## 取消关联

```objective-c
objc_setAssociatedObject(array, &overviewKey, nil, OBJC_ASSOCIATION_ASSIGN);
```
    
policy可以任意设置

另外还可以使用 objc_removeAssociatedObjects，不过这是不被赞成的，因为他打破了所有用户的所有关联。

## 完整的样例代码

```objective-c
#import <objc/runtime.h>
……

static char overviewKey;
NSArray *array = @[@"One", @"Two", @"Three"];
NSString *overview = @"First three numbers";
objc_setAssociatedObject (
        array,
        &overviewKey,
        overview,
        OBJC_ASSOCIATION_RETAIN
        );

NSString *associatedObject =
(NSString *) objc_getAssociatedObject (array, &overviewKey);
NSLog(@"associatedObject: %@", associatedObject);

objc_setAssociatedObject (
        array,
        &overviewKey,
        nil,
        OBJC_ASSOCIATION_ASSIGN
        );
```

注意：本样例代码使用了Objc2.0语法和ARC，更详细的信息，请参考官方文档。
