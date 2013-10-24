---
title: Cocoa代码规范官方指南（要点与简译）
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - cocoa
  - iOS
  - OBJC
  - 代码规范
---
# 背景
我一年前的时候看这篇的时候，对cocoa还处在刚入门阶段，很多细节并没有留下太深刻的影响。如今经常阅读别人的代码了之后，越来越体会到代码规范的重要性，故而重新找回这一篇，又仔细读了一遍，收获颇丰。本着善待别人就是善待自己的原则，就顺手翻译了一遍，希望每个人都有收获，能写规范的代码，让别人看得舒服。

英文好的同学，建议直接看英文，因为很多东西我都不知道怎么翻译，这一篇原名叫 Coding Guidelines for Cocoa

# 命名基础
## 一般原则
### 清楚
* 在保证清楚的前提下才能简化

| Code | Commentary|
|-----|------|
|insertObject: atIndex:|Good.|
|insert:at: | Not clear; what is being inserted? what does “at” signify?|
|removeObjectAtIndex: | Good.|
|removeObject: | Good, because it removes object referred to in argument.|
|remove: | Not clear; what is being removed?|

* 一般来说，不要使用缩写，即使他的全称很长

|Code | Commentary |
|-----|----|
|destinationSelection|Good.|
|destSel|Not clear.|
|setBackgroundColor:|Good.|
|setBkgdColor:|Not clear.|

* 只有一些少数的缩写的确是为绝大多数人所知的，可以继续使用（见附录）
* 避免含糊不清API命名，比如能够以一种以上的方式进行解释的名称

|Code|Commentary|
|-------|-------|
|sendPort|Does it send the port or return it?|
|displayName|Does it display a name or return the receiver’s title in the user interface?|

### 一致性
* 使用和cocoa一致的命名风格，如果你不确定，可以参看头文件和文档
* 一致性在你的类方法使用了多态性的情况下尤为重要，不同类却做相同事的方法应该有相同的名字

| Code | Commentary |
| ---- | ---- |
| - (NSInteger)tag | Defined in NSView, NSCell, NSControl. |
| `- (void)setStringValue:(NSString *)` | Defined in a number of Cocoa classes. |


### 不要自参考（No self-reference）
* 命名不要自参考

| Code | Commentary |
| ---- | ---- |
|NSString | Okay. |
|NSStringObject | Self-referential.|

* 常量是这个原则的例外，比如mask，比如notification，（译者注：比如encoding）

| Code | Commentary |
| ---- | ---- |
| NSUnderlineByWordMask | Okay. |
| NSTableViewColumnDidMoveNotification | Okay. |

## 前缀
因为没有命名空间，所以前缀是很必要的

* 前缀也有指定的格式，两到三个字母的大写缩写，不要使用下划线

| Prefix | Cocoa Framework |
| ---- | ---- |
| NS | Foundation |
| NS | Application Kit |
| AB | Address Book |
| IB | Interface Builder |

* 命名类，协议，函数（译者注：这里是指c-style函数），常量，typedef结构使用前缀。命名类方法时不要使用前缀，因为class已经定义了他们。也不要在命名the fields of structure使用前缀。（译者注：不知道如何翻译，求各路大神指点）

## 命名约定
* 使用驼峰法命名，即不要使用任何分隔符，而使用单词首字符大写的方式来分割。要额外注意下面两点：
    * 方法首字母小写，不要使用前缀

        ```objective-c
      ￼￼fileExistsAtPath:isDirectory:    
        ```
    一个例外，当使用广为人之的缩写的作为方法开头时，比如TIFFRepresentation (NSImage).
    * 定义方法和常量的时候，使用相关前缀并第一个单词首字母大写

        ```objective-c
        NSRunAlertPanel
        NSCellDisabled
        ```
            
* 不要使用下划线来标记私有方法，这是苹果的保留字段。冒然使用会导致不可知的冲突。（译者注：或者上不了架）

## 类和协议的命名
类的名字应该包含一个名词，用来指明这个类（或者这个类的对象）表示什么，或者做了什么。类名需包含一个适当的前缀

协议的命名取决于协议如何组织行为

* 大多数协议将一些相关的方法组合在一起，但是又不关联特定的类，这样的协议命名需要注意不被误认为一个类，通常的做法，是使用动名词形式（-ing）

| Code | Commentary |
| ---- | ---- |
| NSLocking |  Good. |
| NSLock | Poor (seems like a name for a class). |

* 有一些协议将一些不相关的方法组合在一起，却是为了指向特定的类，这时可以将协议命名为于类名相同。一个例子就是NSObject protocol。（译者注：这个情况有点像java里的interface，应用这种协议的方式就能达到相同接口不同实现的效果。虽然我从没见过有人这么做。如果这里我有理解错的地方，各路大神请斧正）

## 头文件
* 声明一个单独的类或协议，比如NSLocale.h 声明 The NSLocale class.
* 声明关联的类和协议，比如NSString.h 声明 NSString and NSMutableString classes；再比如NSLock.h 生命了NSLocking protocol and NSLock, NSConditionLock, and NSRecursiveLock classes.
* 包含框架头文件，比如Foundation.h包含Foundation.framework.
* 添加API到另一个框架的类，If you declare methods in one framework that are in a category on a class in another framework, append “Additions” to the name of the original class; an example is the NSBundleAdditions.h header file of the Application Kit.
* 相关的函数或者类型，If you have a group of related functions, constants, structures, and other data types, put them in an appropriately named header file such as NSGraphics.h (Application Kit).

# 方法命名
## 一般原则
* 首字母小写，驼峰命名法，不要使用前缀
* 对于对某个对象进行操作的方法，以一个动词开头，比如
          
```objective-c
- (void)invokeWithTarget:(id)target;
- (void)selectTabViewItem:(NSTabViewItem *)tabViewItem
```

  不要使用do和does，因为他们没有添加任何说明意义，也不要在动词前加副词和形容词
  
* 如果方法返回一个属性，则可以以属性为开头命名方法名，“get”是不必要的，除非是间接的返回。

```objective-c
- (NSSize)cellSize;  //Right.
- (NSSize)calcCellSize; //Wrong.
- (NSSize)getCellSize; //Wrong.
```
* 在所有参数前使用关键词

```objective-c
- (void)sendAction:(SEL)aSelector to:(id)anObject forAllCells:(BOOL)flag; // Right
- (void)sendAction:(SEL)aSelector :(id)anObject :(BOOL)flag; // Wrong
```
* 使用描述性的关键字描述参数

```objective-c
- (id)viewWithTag:(NSInteger)aTag; //Right.
- (id)taggedView:(int)aTag; // Wrong.
```
* 在子类里面扩展父类已有的方法时，再原来的方法后面添加关键字，比如：

```objective-c
- (id)initWithFrame:(CGRect)frameRect; //NSView, UIView.
- (id)initWithFrame:(NSRect)frameRect
mode:(int)aMode cellClass:(Class)factoryId
numberOfRows:(int)rowsHigh
numberOfColumns:(int)colsWide;
// NSMatrix, a subclass of NSView 
```
* 参数之间不要使用“and”连接

```objective-c
- (int)runModalForDirectory:(NSString *)path file:(NSString *)
name types:(NSArray *)fileTypes;        //Right.
- (int)runModalForDirectory:(NSString *)path andFile:(NSString
*)name andTypes:(NSArray *)fileTypes;   //Wrong.
```

* 当一个方法执行两个不同操作时，使用“and”

```objective-c
- (BOOL)openFile:(NSString *)fullPath withApplication:(NSString *)appName andDeactivate:(BOOL)flag;
```
        
## 属性访问方法
* 如果property表现为一个名词，则

```objective-c
- (type)noun;
- (void)setNoun:(type)aNoun;
```
* 如果property是形容词，则

```objective-c
- (BOOL)isEditable;
- (void)setEditable:(BOOL)flag;
```
* 如果property是动词，则

```objective-c
- (BOOL)verbObject;
- (void)setVerbObject:(BOOL)flag;
```
* 不要把动词分词形式当初形容词用，例如：

```objective-c
- (void)setAcceptsGlyphInfo:(BOOL)flag; //Right.
- (BOOL)acceptsGlyphInfo; //Right.
- (void)setGlyphInfoAccepted:(BOOL)flag; //Wrong.
- (BOOL)glyphInfoAccepted; //Wrong.
```
* 可以使用情态动词（can，show，will等），但是不要使用do，does

```objective-c
- (void)setCanHide:(BOOL)flag; //Right.
- (BOOL)canHide; //Right.
- (void)setShouldCloseDocument:(BOOL)flag; //Right.
- (BOOL)shouldCloseDocument; //Right.
- (void)setDoesAcceptGlyphInfo:(BOOL)flag; //Wrong.
- (BOOL)doesAcceptGlyphInfo; //Wrong.
```
* "get"只用于间接返回对象或数值，你应该只在需要返回多个值的适合使用这样的形式：

```objective-c
- (void)getLineDash:(float *)pattern count:(int *)count phase:(float *)phase;
```
在这样的方法里，你应该允许接受NULL作为参数，当呼叫者不关心多个返回值时。

## 代理方法
* 用能指明发送信息的类的名字作为delegate方法名的开头

```objective-c
- (BOOL)tableView:(NSTableView *)tableView shouldSelectRow:(int)row;
- (BOOL)application:(NSApplication *)sender openFile:(NSString *)filename;
```
* 其他的参数在附在前一条的后面，除非只有一个参数the sender

```objective-c
- (BOOL)applicationOpenUntitledFile:(NSApplication *)sender;
```
* post notification的方法为例外，这时，单个参数为notification对象

```objective-c
- (void)windowDidChangeScreen:(NSNotification *)notification;
```
* 用“did”和“will”来表示某些事已经发生或即将发生

```objective-c
- (void)browserDidScroll:(NSBrowser *)sender;
- (NSUndoManager *)windowWillReturnUndoManager:(NSWindow *)window;
```
* 询问delegate，某事要发生时是否允许，使用should

```objective-c
￼- (BOOL)windowShouldClose:(id)sender;
```
## 容器方法（collection methods）
* 一般形式

```objective-c
- (void)addElement:(elementType)anObj;
- (void)removeElement:(elementType)anObj; 
- (NSArray *)elements;
```
* 如果容器是明显无序的，返回NSSet来替代NSArray。
* 插入和删除

```objective-c
- (void)insertLayoutManager:(NSLayoutManager *)obj atIndex:(int)index;
- (void)removeLayoutManagerAtIndex:(int)index;
```
* 实现细节上的注意事项：
    * 一般来说，插入的对象包涵所有关系，所以在你插入对象的时候，应该retain他们，删掉他们的时候，应该release。
    * If the inserted objects need to have a pointer back to the main object, you do this (typically) with a set... method that sets the back pointer but does not retain. In the case of the insertLayoutManager:atIndex: method, the NSLayoutManager class does this in these methods:
```objective-c
- (void)setTextStorage:(NSTextStorage *)textStorage;
- (NSTextStorage *)textStorage;
```
You would normally not call setTextStorage: directly, but might want to override it. (译者注：这个情况我也不是很熟悉，为避免让人误解，故保留原文，有大神准确的明白这里指什么的，请指教)

## 方法参数
对于方法的参数的命名，有一般原则：

* 小写开头的驼峰命名(for example, removeObject:(id)anObject).
* 不要使用“pointer”或“ptr”，让参数的类型来说明这些
* 避免一个火两个字母的参数
* 避免使用不必要的缩写

cocoa的惯例命名如下：

```objective-c
￼...action:(SEL)aSelector
...alignment:(int)mode
...atIndex:(int)index
...content:(NSRect)aRect
...doubleValue:(double)aDouble
...floatValue:(float)aFloat
...font:(NSFont *)fontObj
...frame:(NSRect)frameRect
...intValue:(int)anInt
...keyEquivalent:(NSString *)charCode
...length:(int)numBytes
...point:(NSPoint)aPoint
...stringValue:(NSString *)aString
...tag:(int)anInt
...target:(id)anObject
...title:(NSString *)aString
```
    
## 私有方法
大部分情况下，私有方法的命名和共有方法一样，然而，一个常见的惯例是给私有方法前加一个前缀来标示这是一个私有方法。Coca frameworks里使用下划线前缀来标示私有。因此，

* 不要使用下划线作为前缀来标示你自己实现的私有方法，苹果已经保留了这种形式
* 如果你继承了一个Cocoa framework class，为了确保你的私有方法与其不一样，你应该加上你自己的前缀。前缀应该尽量唯一化，一般来说为“XX_”，XX可以是你的公司或项目缩写。

虽说这里使用前缀违反了前面说的，类方法不要使用前缀，而是由类本身作为命名空间的原则，但，在这里的意图不一样，这里的意图是为了避免非故意的重写父类的私有方法。
（译者注：OBJC没有真正意义上的私有，即使是使用了如上这样的标示方法，也只是提醒他人这个函数是私有函数，并没有强制的保证别人无法访问。另一种常用的标示私有的做法，是把私有方法的声明放在m文件，而不是h文件）

# 函数命名
一般原则

* 函数的命名和方法相似，不过又一些例外
    * 使用和类名或常量相同的前缀
    * 前缀后第一个字母大写
* 大多数函数以动词开头来描述函数的功能：
```objective-c
NSHighlightRect
NSDeallocateObject
```
请求属性的函数，有另一些规则
* 如果函数返回一个来自其第一个参数的属性，则省略动词
```objective-c
unsigned int NSEventMaskFromType(NSEventType type)
float NSHeight(NSRect aRect)
```
* 如果返回的值是引用，则使用“get”
```objective-c
const char *NSGetSizeAndAlignment(const char *typePtr, unsigned int *sizep, unsigned int *alignp)
```
* 如果返回的值是bool，
```objective-c
￼￼BOOL NSDecimalIsNotANumber(const NSDecimal *decimal)
```
        
# 属性和数据类型命名
## 属性和实例变量
一般来说和属性访问方法的原则相同，例如：
```objective-c
@property (strong) NSString *title;
@property (assign) BOOL showsAlpha;
```
如果是形容词，可以这样
```objective-c
@property (assign, getter=isEditable) BOOL editable;
```

变量命名需简明的描述储存的属性。一般来说，你不应该直接访问变量，而是使用访问方法，（你只在init和dealloc里面直接访问变量）。

There are a few considerations to keep in mind when adding instance variables to a class:
* Avoid explicitly declaring public instance variables.
Developers should concern themselves with an object’s interface, not with the details of how it stores its data. You can avoid declaring instance variables explicitly by using declared properties and synthesizing the corresponding instance variable.
* If you need to declare an instance variable, explicitly declare it with either @private or @protected.
If you expect that your class will be subclassed, and that these subclasses will require direct access to the
data, use the @protected directive.
* If an instance variable is to be an accessible attribute of instances of the class, make sure you write accessor
methods for it (when possible, use declared properties).
       
（译者注：在这部分上我持保留意见，这应该不是现代的做法，事实上，我还没有见过有人使用@private，@protected，我自己尝试在代码里面使用这个关键词却被编译器报错。为避免引起人误解，这里保留原文，欢迎大家讨论）

## 常量
### 枚举常量
* 用枚举来给组合一组相关的常量，其类型通常为int
* 使用和函数命名相同的规则，如
```objective-c
typedef enum _NSMatrixMode {
    NSRadioModeMatrix = 0，
    NSHighlightModeMatrix = 1，
    NSListModeMatrix = 2，
    NSTrackModeMatrix = 3
} NSMatrixMode;
```
注意这里的 typedef 标签并不是必须的。
* 可以创建没有命名的枚举
```objective-c
enum {
    NSBorderlessWindowMask      = 0,
    NSTitledWindowMask          = 1 << 0,
    NSClosableWindowMask        = 1 << 1,
    NSMiniaturizableWindowMask  = 1 << 2,
    NSResizableWindowMask       = 1 << 3
};
```
### const常量
* Use const to create constants for floating point values. You can use const to create an integer constant if the constant is unrelated to other constants; otherwise, use enumeration.
* 使用和函数命名相同的规则
### 其他类型的常量
* 一般来说不要使用 #define 来创建常量
* 使用大写字母来进行预处理标示，判断一个代码块是否需要处理，比如
```objective-c
￼#ifdef DEBUG
```
* macro使用两个下划线前缀和后缀，比如：
```objective-c
__MACH__
```
* Define constants for strings used for such purposes as notification names and dictionary keys. By using string constants, you are ensuring that the compiler verifies the proper value is specified (that is, it performs spellchecking).TheCocoa frameworks provide many examples of string constants, such as:
    ```objective-c
    APPKIT_EXTERN NSString *NSPrintCopies;
    ```
The actual NSString value is assigned to the constant in an implementation file. (Note that the APPKIT_EXTERN macro evaluates to extern for Objective-C.)
(译者注：我没法使用APPKIT_EXTERN，查看了一下cocoa头文件，发现其使用的是FOUNDATION_EXPORT，为避免引起误解，这里也保留原文)


## 通知和异常
### 通知
Global NSString objects
```objective-c
[Name of associated class] + [Did | Will] + [UniquePartOfName] + Notification
```

例如：

```objective-c
NSApplicationDidBecomeActiveNotification
NSWindowDidMiniaturizeNotification
NSTextViewDidChangeSelectionNotification
NSColorPanelColorDidChangeNotification
```
### 异常
Global NSString objecsts

```objective-c
[Prefix] + [UniquePartOfName] + Exception
```
    
例如：

```objective-c
NSColorListIOException
NSColorListNotEditableException
NSDraggingException
NSFontUnavailableException
NSIllegalSelectorException
```
    
# 接受的缩写
| Abbreviation | Meaning and comments |
| ---- | ---- |
| alloc | Allocate. |
| alt | Alternate. |
| app | Application. For example, NSApp the global application object. However, “application” is spelled out in delegate methods, notifications, and so on. |
| calc | Calculate. |
| dealloc | Deallocate. |
| func | Function. |
| horiz | Horizontal. |
| info | Information. |
| init | Initialize (for methods that initialize new objects). |
| int | Integer (in the context of a C int—for an NSInteger value, use integer). |
| max | Maximum. |
| min | Minimum. |
| msg | Message. |
| nib | Interface Builder archive. 
| pboard | Pasteboard (but only in constants). |
| rect | Rectangle. |
| Rep | Representation (used in class name such as NSBitmapImageRep). |
| temp | Temporary. |
| vert | Vertical. |
接受的大写缩写词：
ASCII PDF XML HTML URL RTF HTTP TIFF JPG PNG GIF LZW ROM RGB CMYK MIDI FTP

# Tips and Techniques for Framework Developers
frameworks的开发者需要比其他的开发者更小心。许多用户应用都可能会连接到这些framework，因正因为如此framework的任何都不足，都将被放大。接下来描述的技术可以用于保证你的框架的效率和可靠性。
注意：其中的一些技术不单单适于用框架，你也可以用于应用开发。
（译者注：这一节比较复杂，因为我没开发过frameworks，所以就不冒然翻译了，待我研究清楚后，另开一篇详说。有兴趣的，也可以自行查看原文）
# Contact Me
* [Follow my github](https://github.com/lancy)
* [Follow my weibo](http://weibo.com/lancy1014)
* Send Email to me: lancy1014@gmail.com
