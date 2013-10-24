---
title: Objective-C Associative References(关联引用) 续：相关实践
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - cocoa
  - iOS
  - OBJC
  - UIKit
  - 关联引用
---

## About
我之前写了一篇博文[Objective-C Associative References(关联引用)](http://gracelancy.com/?p=82)，介绍我在在研究objc runtime的有趣的发现，但当时我并没有意识到这个技术应该使用在何处。在一些实践之后，小结一下有关关联引用的一些相关实践吧。

## Category中使用关联引用来添加property

我们知道category是不能创建实例变量的，但我们可以通过关联引用来达到这样的目的。特别是当你不持有这个类，比如说系统的类，而你又的确需要添加一个property。

你可以这样做：

```objective-c
#import <objc/runtime.h>

@interface Person (EmailAddress)
@property (readwrite, copy) NSString *emailAddress;
@end

@implementation Person (EmailAddress)
static char emailAddressKey;
- (NSString *)emailAddress {
    return objc_getAssociatedObject(self, &emailAddressKey);
}

- (void)setEmailAddress:(NSString *)emailAddress {
        objc_setAssociatedObject(self, &emailAddressKey, emailAddress, OBJC_ASSOCIATION_COPY);
} 

@end
```

## 给UI控件关联上相关对象

比如UIAlert只有一个tag属性用来做标记，我们经常需要根据Tag属性在找出对应需要操作的对象。但使用关联对象，我们可以把UIAlert和某个对象关联，简化这个过程。

比如你可以这样做：

```objective-c
id interestingObject = ...;
UIAlertView *alert = [[UIAlertView alloc]
                     initWithTitle:@”Alert” message:nil
                     delegate:self
                     cancelButtonTitle:@”OK”
                     otherButtonTitles:nil];
objc_setAssociatedObject(alert, &kRepresentedObject,
                        interestingObject,
                     OBJC_ASSOCIATION_RETAIN_NONATOMIC);
[alert show];
```
         
在alertView的delegate方法里面这样操作:

```objective-c
- (void)alertView:(UIAlertView *)alertView
clickedButtonAtIndex:(NSInteger)buttonIndex {
    UIButton *sender = objc_getAssociatedObject(alertView, &kRepresentedObject);
    self.buttonLabel.text = [[sender titleLabel] text];
}
```

## 结合以上两者的最佳实践
在Cocoa里面，我们经常会见到user info这样一个属性，（比如NSNotification.userinfo），代表用户自定义的payload数据。

同时一般而言，显式的使用objc的runtime特性并不是一个良好的编程习惯，故而我们可以使用category给UIAlert添加一个user info的property，以将objc的runtime代码进行隐藏。

代码与前面给出的类似，你可以在Github下载到完整Demo。
[传送门](https://github.com/lancy/UIAlertViewUserinfo)

使用效果：

```objective-c
UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Alert One" message:@"I gonna show the userinfo" delegate:self cancelButtonTitle:@"OK" otherButtonTitles: nil];
[alert setUserinfo:@{@"message": @"I'm userinfo of alert one"}];
[alert show];
```
