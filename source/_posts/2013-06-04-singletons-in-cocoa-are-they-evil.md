---
title: Singletons in Cocoa, are they evil?
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - cocoa
  - design pattern
  - iOS
  - OBJC
  - singleton
---
## 故事
这事是这样的，去年我在上课的时候，和老师讨论了一下关于架构的问题，我是开发Cocoa/iOS的，老师是开发Web的，而老师是一个坚定的singletons are evil的拥护者，我和他说了我的App的架构，直接被他一顿猛劈，强烈的谴责了我使用Singletons，我回应说，这个pattern在Cocoa里是大量使用的，结果被搞了一句“用的多的就是对的么？你回去多学习一下再来讨论吧”。

于是我非常郁闷的回去搜索的一大顿的资料，还在Stackoverflow上发起了一个问题：[singletons in cocoa, are they evil?](http://stackoverflow.com/questions/13306268/singletons-in-cocoa-are-they-evil)。甚至在某个社区，假扮singleton are evil的拥护者，把所有singleton的缺点列了一堆，结果又是群起而攻之一场舌战。

关于Singleton的缺点，放出一段引用：
> 1. They are generally used as a global instance, why is that so bad? Because you hide the dependencies of your application in your code, instead of exposing them through the interfaces. Making something global to avoid passing it around is a code smell.
> 
> 2. They violate the Single Responsibility Principle: by virtue of the fact that they control their own creation and lifecycle.
> 
> 3. They inherently cause code to be tightly coupled. This makes faking them out under test rather difficult in many cases.
> 
> 4. They carry state around for the lifetime of the app. Another hit to testing since you can end up with a situation where tests need to be ordered which is a big no no for unit tests. Why? Because each unit test should be independent from the other.


公说公有理，婆说婆有理，一度把我弄得越来越困惑，后来我看到这一段话，我就彻底释然了：
> As for degrees of evil - it's like speech or literature. F-words are "evil". If you speak constantly using f-words words the quality of your language is lower - people can't tell if you want to say something or just swearing. But in some situations such words help you to get things done (think of the battlefield orders). It sort of the same thing with features/patterns and people who have to read and maintain their usage. 
> 
> – hoha 

BTW，今天我甚至看到了[Accessors Are Evil](http://c2.com/cgi/wiki?AccessorsAreEvil)这样的东西，更坚定了我再也不相信xxx are evil这种说法的决心。

我现在认为Design pattern是前人总结的经验，不同的设计模式有不同的优缺点，比如说用工厂代替单例的，虽说解决了单例的一些问题，但你要真去写一个工厂就知道有多蛋疼，多浪费生命了。然而在较为大型的应用，非常多人协作的项目，队友对项目的把握不一致，水平有高低之分，这时工厂又反而是一种安全的，省时省力的做法。

其实在代码的世界里面，你想要更多的安全，就会丧失更多的灵活性和便利性。如何在这中间取舍，就需要我们彻底的了解某种模式（或者说某种编程方法）的优缺点，在保证基本的安全性的情况下，尽可能的减少工作量，提高工作效率。

## Singletons in Cocoa
回到正题，还是来说说Cocoa上的单例。Cocoa中的普遍的，大部分的单例，并不是严格的单例（strict singleton），而是一种共享单例（shared singleton），例如sharedApplication，sharedURLCache等。即，大多数情况，我们访问同一个类方法，就可以获得一个同样的实例，但若真的需要存在多个实例亦可。通常，共享单例使用一个shared开的类方法识别。只有当真的只有唯一的一个共享资源的时候，或者不可能有多个资源的时候（比如GPS模块），才会使用严格意义的共享单例。

## 线程安全的Singleton
绝大多数情况下，使用一个共享单例比使用共享单例要好，然而这里有一个常见的创建共享单例的错误，即使是Apple自己的开发者文档也没弄清楚的一个错误，他们把Singleton写成了非线程安全的：

```objective-c
+ (MyClass *)sharedInstance {
    static MyClass *sharedInstance;
    if (sharedInstance == nil) {
        sharedInstance = [[MyClass alloc] init];
    }
    return sharedInstance;
}
```

正确的写法应该是：

```objective-c
+ (MyClass *)sharedInstance {
    static MyClass *sharedInstance;
    @synchronized(self) {
        if (sharedInstance == nil) {
            sharedInstance = [[MyClass alloc] init];
        }
    }
    return sharedInstance;
}
```    
更恰当的写法是使用dispatch_once()

```objective-c
+ (MYClass *)sharedInstance
{
    static dispatch_once_t pred = 0;
    static MYClass _sharedObject = nil;
    dispatch_once(&pred, ^{
            _sharedObject = [[self alloc] init]; // or some other init method
            });
    return _sharedObject;
}
```

dispatch_once()即为执行且仅仅执行某个block一次，他是同步的方法（记住GCD也有很多同步的方法），其速度也比 @synchronized 快许多。

## 严格的单例(strict singleton)
尽管我们很少会使用到严格的单例模式，但当真的需要的时候，还是可以实现的。

苹果官方文档提供了一个严格单例的实现（[传送门](http://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CocoaFundamentals/CocoaObjects/CocoaObjects.html#//apple_ref/doc/uid/TP40002974-CH4-SW32)）。
其重载了allocWithZone:, copyWithZone, retain, retainCount, release, autorelease。使得这个实现变得无比复杂而难以理解和控制。

而大多数情况下，实现严格的单例模式，只需要和共享单例相同的代码，再使用NSAssert使得一切调用init的代码作为一个错误处理即可，代码如下：

```objective-c
+ (MYSingleton *)sharedSingleton {
     static dispatch_once_t pred;
     static MYSingleton *instance = nil;
     dispatch_once(&pred, ^{instance = [[self alloc] initSingleton];});
     return instance;
}
- (id)init {
    // Forbid calls to –init or +new
    NSAssert(NO, @”Cannot create instance of Singleton”);
    // You can return nil or [self initSingleton] here, 
    // depending on how you prefer to fail.
    return nil;
}
// Real (private) init method
- (id)initSingleton {
    self = [super init];
    if ((self = [super init])) {
    // Init code }
    return self; 
}
```

这份代码的优点是很明显的，避免了复杂的内存操作和重载，又静止了调用者创建多个实例。

## 小结
小结一下，单例模式是Cocoa中非常常用的一个模式，对于应用程序中广泛使用的对象，单例模式是非常便利的方法。而我们也应当在使用的时候多注意单例模式的一些缺点，尽可能的在实现的时候避免他们，比如让单例不存在过于复杂的依赖性和继承，保证其松耦合等。

## Edit:
One more thing:有筒子问到是@synchronized(self)还是@synchronized(sharedInstance)?

答案是：均可。

self，在实例方法中表现是实例，这一点自不用多说。在类方法中则表现为一种多态的类实例（class instance），他总是会返回正确的类型，比如这样：

```objective-c
+ (id)new
{
    return [[self alloc] init];
}
```

而在本文的这个@synchronized(self)里的self，总是会指向同一个对象，即那个特殊的类实例。（class也是一个对象），故而此处可以使用self。

lancy
