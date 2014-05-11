---
layout: post
title: "Variable Argument Lists"
date: 2014-05-05 00:11
comments: true 
categories:
  - iOS
tags:
  - cocoa
  - iOS
  - OBJC
  - Variable argument lists
---

## Variable argument lists 使用方法
可变参数函数（Variadic Function），即是指一个可以接受可变数量的参数的函数。在C语言中，对该特性的支持，即是通过可变参数列表（Variable Argument list）来实现的，其定义在`stdarg.h`头文件。(若使用C++则在`cstdarg`头文件)。

以如下C代码为例说明，该函数接受可变数量的整数作为参数，求和：

```c
int addemUp (int firstNum, ...) {
	// 1. 参数后面添加省略号...
    va_list args;  // 2. 创建一个va_list类型的变量
    int sum = firstNum;
    int number;
    va_start(args, firstNum); // 3. 初始化va_list，此时va_list指向firstNum之后的第一个参数
    while (1) {
        number = va_arg(args, int); // 4. 获取当前指向的参数的值，并移动到下一个参数
        sum += number;
        if (number == 0) { 
        	// 用0表示结束
            break;
        }
    }
    va_end(args); // 5. 清理
    return  sum;
}
// 调用
sum = addemUp(1,2,3,4,5,0);
// sum = 15
```

1. 要创建一个可变参数函数，需要把一个省略号（...)放在函数的参数列表后面。
2. 接着需要声明一个一个`va_list`类型的变量，这个`va_list`类型的变量类似于一个指向参数的指针。
3. 接着我们调用`va_start()`并传入函数的最后一个声明的参数的变量名，来使得`va_list`变量指向第一个附加的参数。
4. 接着我们调用`va_arg()`并传入我们期待的参数类型，程序就会返回与该类型匹配数量的字节（即参数的值），并且移动`va_list`指向下一个参数。之后不断的调用`va_arg()`，获得更多的参数的值，直到完成整个参数处理的过程。
5. 最后调用`va_end()`来进行清理。

## variable argument lists 的内部机制
如我们之前所说，当我们调用`va_start()`并将`va_list`和函数最后定义的参数传入时，实际上是将`va_list`内在的一个指针指向函数调用栈 （[call stack](http://en.wikipedia.org/wiki/Call_stack#Functions_of_the_call_stack)）中参数所在的区域的一端，每一次我们调用`va_arg()`，其都会根据提供的类型，返回当前指针所指向的地址开始对应的字节数的数据，即参数的值，并移动指针相应字节数的距离。我们传给`va_arg()`的类型，即是其用来判定需要取得得数据的大小，以及指针需要移动的距离。如图描述了这个过程：

<img src="{{ size.url }}/assets/post/val0.png" alt="call stack" style="width: 600px;"/>

事实上，这是一个很危险的事情，你总是需要提供正确的类型来让`va_arg()`正确执行，而且`va_arg()`并不知道何时停止，你需要提供一个标记或一个参数的总数来停止`va_arg()`继续执行。若你提供了不正确的类型，或者没有在该停止的时候停止，你将会获得不可预测的值，并且很有可能导致程序崩溃。

## 解决方案
一般而言，为了确保参数的获取正确进行，有如下两种解决方案：
### Format string
如C语言中的`printf`，Cocoa中的`NSLog`，`[NSString stringWithFormat:]`就是使用了Format String的解决方案。通常，该函数的第一个参数既为一个format string，函数内部实现会扫描这个format string，来确定之后接着的可变参数的数量和类型。例如：
```objc
NSString *str = [NSString stringWithFormat:@"int %d, str %@, float $g", 123, @"ok", 123.4];
```
这里使用了@作为转义符，其后跟着的d代表int，@代表id，g代表float/double，这表示后面必须有三个参数，其类型必须与format string所指定的一致。

如之前所说，提供的参数的数量或者类型若与提供的format string不一致，则会发生不可预知的问题。而在运行的时候，我们没有任何的办法去保证其正确性，幸运的是编译器提供了一些方法，能让我们在编译的时候做一些检查：

gcc中定义了`__attribute__((format))`来标示一个可变参函数使用了format string，从而在编译时对其进行检查。其定义为`format (archetype, string-index, first-to-check)`，其中`archetype`代表format string的类型，它可以是`printf`，`scanf`，`strftime`或者`strfmon`，Cocoa开发者还可以使用`__NSString__`来指定其使用和`[NSString stringWithFormat:]`与`NSLog()`一致的format string规则。`string-index`代表format string是第几个参数，`first-to-check`则代表了可变参数列表从第几个参数开始。示例：

```objc
// 第一个参数是format，第二个参数起是可变参数列表，format的格式规则与printf一致
void customPrintf(const char *format, ...) __attribute__((format(printf, 1, 2)));

// 使用的时候，若format和参数不符，则会报warning
customPrintf("what? %d", 1.2, 2);
```

Cocoa开发者可以使用`NS_FORMAT_FUNCTION(F,A)`宏来替代`__atribute__format`，F和A即对应`string-index`和`first-to-check`，事实上，他的实现类似于：
```c
#define NS_FORMAT_FUNCTION(F,A) __attribute__((format(__NSString__, F, A)))
```

示例如下：

```objc
FOUNDATION_EXPORT void NSLog(NSString *format, ...) NS_FORMAT_FUNCTION(1,2);
```

### Sentinel value
哨兵值是另一种可变参数列表所常用的方案，如前一节我们的示例代码，即是使用了数字0作为哨兵值。当程序发现当前读取到的参数值为0时，则停止继续读取程序。在Cocoa中，我们经常使用`nil`作为哨兵值，比如`[NSArray arrayWithObjects:]`方法，其接受数量不等的对象作为参数，而在最后则必须使用nil结尾。如：
```objc
[NSArray arrayWithObjects:@1, @2, @3, nil];
//备注：我们现在通常使用@[@1, @2, @3]来代替这一行代码，且不需要在最后添加nil，这称为字面量（Literals)
```

同format string一样危险的是，若开发者调用方法（函数）的时候，忘记在最后添加上哨兵值，则会发生不可预知的问题。同样幸运的是，编译器也为我们提供了一些方法来在编译时进行检查。

gcc中定义了`___attribute__((sentinel))`来标示一个函数需要在编译的时候对哨兵值进行检查。用法如下：

```c
int addemUp (int firstNum, ...) __attribute__((sentinel));
```

Cocoa开发者可以使用`NS_REQUIRES_NIL_TERMINATION`宏来替代，其实现基本等同于上述代码：

```objc
+ (instancetype)arrayWithObjects:(id)firstObj, ... NS_REQUIRES_NIL_TERMINATION;
```

## 工程实例
我在开发猿题库iOS客户端时，由于产品的需要会有许多alert弹框。但传统的`UIAlertView`经常需要实现相应的`UIAlertViewDelegate`，使用起来非常不便。我写了一个能够接收block作为回调的自定义的AlertView组件，同时为了保证其接口与`UIAlertView`基本一致，使用了可变参数列表。其接口定义如下：

```objc
@interface CYAlertView : UIAlertView

- (id)initWithTitle:(NSString *)title
            message:(NSString *)message
         clickedBlock:(void (^)(CYAlertView *alertView, BOOL cancelled, NSInteger buttonIndex))clickedBlock
  cancelButtonTitle:(NSString *)cancelButtonTitle
  otherButtonTitles:(NSString *)otherButtonTitles, ... NS_REQUIRES_NIL_TERMINATION;

@end
```

完整的代码开源托管在GitHub（[传送门](https://github.com/lancy/cyalertview)），有兴趣的同学可以参考。

## 联系我

水平有限，若有任何关于该文章的疑问或者指正，欢迎和我讨论

* 写邮件：lancy1014#gmail.com
* 关注我的[微博](http://weibo.com/lancy1014/)
* Fo我的[Github](http://github.com/lancy)
* 在这里写评论留言

## 参考
* [Clang 3.5 documentation: Attributes in Clang](http://clang.llvm.org/docs/AttributeReference.html#format-gnu-format)
* [GCC documentation: Function Attributes](http://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html)
* [NSHisper: __attribute__ ](http://nshipster.com/__attribute__/)
* Advanced Mac OS X Programming

lancy

2014.5.12