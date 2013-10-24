# Thinking in Objective-C: Cocoa Design patterns

## 
As a general rule, objects do not retain their delegates. If you create a class with a delegate property, it should almost always be declared weak. In most cases, an object’s delegate is also its controller, and the controller almost always retains the original object. If the object retained its delegate, you would have a retain loop and would leak memory. There are exceptions to this rule. For example, NSURLConnection retains its delegate, but only while the connection is loading. After that NSURLConnection releases its delegate, avoiding a permanent retain loop.

[参考链接](http://developer.apple.com/library/mac/#documentation/Cocoa/Reference/Foundation/Classes/NSURLConnection_Class/Reference/Reference.html)


Note: During a download the connection maintains a strong reference to the delegate. It releases that strong reference when the connection finishes loading, fails, or is canceled.

## 

The placement of const is important when declaring string constants. This declaration is correct: 

    extern NSString * const RNFooDidCompleteNotification;This declaration is incorrect:
    extern const NSString * RNFooDidCompleteNotification;The former is a constant pointer to an immutable string. The latter is a changeable pointer to an immutable string. NSString is always immutable because it is an immutable class. So NSString * const is useful. const NSString * is useless. This is easier to remember if you read the declaration from right to left: “const pointer to NSString.”
