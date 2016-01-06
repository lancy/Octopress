---
title: 猿题库 iOS 客户端架构设计
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

## 序
猿题库是一个拥有数千万用户的创业公司，从2013年题库项目起步到2015年，团队保持了极高的生产效率，使我们的产品完成了五个大版本和数十个小版本的高速迭代。在如此快速的开发过程中，如何保证代码的质量，降低后期维护的成本，以及为项目越来越快的版本迭代速度提供支持，成为了我们关注的重要问题。这篇文章将阐明我们在猿题库 iOS 客户端的架构设计。

## MVC
MVC，Model-View-Controller，我们从这个古老而经典的设计模式入手。采用 MVC 这个架构的最大的优点在于其概念简单，易于理解，几乎任何一个程序员都会有所了解，几乎每一所计算机院校都教过相关的知识。而在 iOS 客户端开发中，MVC 作为官方推荐的主流架构，不但 SDK 已经为我们实现好了 UIView、UIViewController 等相关的组件，更是有大量的文档和范例供我们参考学习，可以说是一种非常通用而成熟的架构设计。

但 MVC 也有他的坏处。由于 MVC 的概念过于简单朴素，已经越来越难以适应如今客户端的需求，大量的代码逻辑在 MVC 中并没有定义得很清楚究竟应该放在什么地方，导致他们很容易就会堆积在 Controller 里，成为了人们所说的 Massive View Controller。

## MVVM
MVVM，Model-View-ViewModel，一个从 MVC 模式中进化而来的设计模式，最早于2005年被微软的 WPF 和 Silverlight 的架构师 John Gossman 提出。在 iOS 开发中实践 MVVM 的话，通常会把大量原来放在 ViewController 里的视图逻辑和数据逻辑移到 ViewModel 里，从而有效的减轻了 ViewController 的负担。另外通过分离出来的 ViewModel 获得了更好的测试性，我们可以针对 ViewModel 来测试，解决了界面元素难于测试的问题。MVVM 通常还会和一个强大的绑定机制一同工作，一旦 ViewModel 所对应的 Model 发生变化时，ViewModel 的属性也会发生变化，而相对应的 View 也随即产生变化。

同样的，MVVM 也有他的缺点：

一个首要的缺点是，MVVM 的学习成本和开发成本都很高。MVVM 是一个年轻的设计模式，大多数人对他的了解都不如 MVC 熟悉，基于绑定机制来进行编程需要一定的学习才能较好的上手。同时在 iOS 客户端开发中，并没有现成的绑定机制可以使用，要么使用 KVO，要么引入类似 ReactiveCocoa 这样的第三方库，使得学习成本和开发成本进一步提高。

另一个缺点是，数据绑定使 Debug 变得更难了。数据绑定使程序异常能快速的传递到其他位置，在界面上发现的 Bug 有可能是由 ViewModel 造成的，也有可能是由 Model 层造成的，传递链越长，对 Bug 的定位就越困难。

同时还必须指出的是，在传统的 MVVM 架构中，ViewModel 依然承载的大量的逻辑，包括业务逻辑，界面逻辑，数据存储和网络相关，使得 ViewModel 仍然有可能变得和 MVC 中 ViewController 一样臃肿。

## 在两种架构中权衡而产生的架构
两种架构的优点都想要，缺点又都想避开，我们在两种架构中权衡了他们的优缺点，设计出了一个新的架构，起了一个名字叫：MVVM without Binding with DataController，架构图如下：

<img src="{{ size.url }}/assets/post/arch0.png" alt="Drawing" style="width: 640px;"/>

### ViewModel
先来看右边视图相关的部分，传统的 MVC 当中 ViewController 中有大量的数据展示和样式定制的逻辑，我们引入 MVVM 中 ViewModel 的概念，将这部分视图逻辑移到了 ViewModel 当中。在这个设计中，每一个 View 都会有一个对应的 ViewModel，其包含了这个 View 数据展示和样式定制所需要的所有数据。同时，我们不引入双向绑定机制或者观察机制，而是通过传统的代理回调或是通知来将 UI 事件传递给外界。而 ViewController 只需要生成一个 ViewModel 并把这个装配给对应的 View，并接受相应的 UI 事件即可。

这样做有几个好处：首先是 View 的完全解耦合，对于 View 来说，只需要确定好相应的 ViewModel 和 UI 事件的回调接口即可与 Model 层完全隔离；而 ViewController 可以避免与 View 的具体表现打交道，这部分职责被转交给了 ViewModel，有效的减轻了 ViewController 的负担；同时我们弃用了传统绑定机制，使用了传统的易于理解的回调机制来传递 UI 事件，降低了学习成本，同时使得数据的流入和流出变得易于观察和控制，降低了维护了调适的成本。

### DataController
接下来我们关注 Model 和 VC 之间的关系。如之前提到，在传统的 MVVM 中，ViewModel 接管了 ViewController 的大部分职责，包括数据获取，处理，加工等等，导致其很有可能变得臃肿。我们将这部分逻辑抽离出来，引入一个新的部件，DataController。

ViewController 可以向 DataController 请求获取或是操作数据，也可以将一些事件传递给 DataController，这些事件可以是 UI 事件触发的。DataController 在收到这些请求后，再向 Model 层获取或是更新数据，最后再将得到的数据加工成 ViewController 最终需要的数据返回。

这样做之后，使得数据相关的逻辑解耦合，数据的获取、修改、加工都放在 Data Controller 中处理，View Controller 不关心数据如何获得，如何处理，Data Controller 也不关心界面如何展示，如何交互。同时 Data Controller 因为完全和界面无关，所以可以有更好的测试性和复用性。

DataController 层和 Model 层之间的界限并不是僵硬的，但需要保证每一个 ViewController 都有一个对应的 DataController。Data Controller 更强调的是其作为业务逻辑对外的接口。而在 DataController 中调用更底层的 Model 层逻辑是我们推荐的编程范式，例如数据加工层，网络层，持久层等。

在后面的例子中，我们会更详细的讲解 DataController 的实现细节。

## Show me the code
我们以猿题库主页为例，展示我们是如何使用应用这个架构的。

<img src="{{ size.url }}/assets/post/arch1.png" alt="Drawing" style="width: 320px;"/>

主页有几个部分组成，最上面的小猴子 Banner 页，用于滚动展示一些活动信息；中间有一个用户名字的页面，用于展示用户信息和答题情况以及一些心灵鸡汤；最底下的这部分是一个课目选择页面，展示了用户开启的科目入口，在更多选项里面可以进一步配置这些科目入口。接下来我们会以科目页面（SubjectView）为例展示一些细节。

### ViewController
我们会给每一个 ViewController 都创建一个对应的 DataController。
例如我们给主页建一个类起名叫`APEHomePraticeViewController`，同时他会有一个对应的 DataController 起名叫 `APEHomePraticeDataController`。同时我们把页面拆分为几个部分，每个部分有一个相对应的 SubView。代码如下：

```
@interface APEHomePracticeViewController () <APEHomePracticeSubjectsViewDelegate>
	
@property (nonatomic, strong, nullable) UIScrollView *contentView;

@property (nonatomic, strong, nullable) APEHomePracticeBannerView *bannerView;
@property (nonatomic, strong, nullable) APEHomePracticeActivityView *activityView;
@property (nonatomic, strong, nullable) APEHomePracticeSubjectsView *subjectsView;
	
@property (nonatomic, strong, nullable) APEHomePracticeDataController *dataController;
	
@end

```

在 `viewDidLoad` 的时候，初始化好各个 SubView，并设置好布局：

```
- (void)setupContentView {
    self.contentView = [[UIScrollView alloc] init];
    [self.view addSubview:self.contentView];

    self.bannerView = [[APEHomePracticeBannerView alloc] init];
    self.activityView = [[APEHomePracticeActivityView alloc] init];
    self.subjectsView = [[APEHomePracticeSubjectsView alloc] init];
    self.subjectsView.delegate = self;

    [self.contentView addSubview:self.bannerView];
    [self.contentView addSubview:self.activityView];
    [self.contentView addSubview:self.subjectsView];
    // Layout Views ...
}

```
接下来，ViewController 会向 DataController 请求 Subject 相关的数据，并在请求完成后，用获得的数据生成 ViewModel，将其装配给 SubjectView，完成界面渲染，代码如下：

```
- (void)fetchSubjectData {
    [self.dataController requestSubjectDataWithCallback:^(NSError *error) {
        if (error == nil) {
            [self renderSubjectView];
		}
	}];
}
- (void)renderSubjectView {
    APEHomePracticeSubjectsViewModel *viewModel =
        [APEHomePracticeSubjectsViewModel viewModelWithSubjects:self.dataController.openSubjects];
    [self.subjectsView bindDataWithViewModel:viewModel];
}
```
### 数据结构
为了更好的演示，我们接下来要介绍一下 Subject 相关的数据结构：

`APESubject` 是科目的资源结构，包含了 Subject 的 id 和 name 等资源属性，这部分属性是用户无关的；`APEUserSubject` 是用户的科目信息，包含了用户是否打开某个学科的属性。

```
@interface APESubject : NSObject

@property (nonatomic, strong, nullable) NSNumber *id;
@property (nonatomic, strong, nullable) NSString *name;

@end

@interface APEUserSubject : NSObject

@property (nonatomic, strong, nullable) NSNumber *id;
@property (nonatomic, strong, nullable) NSNumber *updatedTime;
///  On or Off
@property (nonatomic) APEUserSubjectStatus status;

@end

```

### DataController
如我们之前所说，每一个 ViewController 都会有一个对应的 DataController，这一类 DataController 的主要职责是处理这个页面上的所有数据相关的逻辑，我们称其为 View Related Data Controller。

```
// APEHomePracticeDataController.h
@interface APEHomePracticeDataController : APEBaseDataController
// 1
@property (nonatomic, strong, nonnull, readonly) NSArray<APESubject *> *openSubjects;
// 2
- (void)requestSubjectDataWithCallback:(nonnull APECompletionCallback)callback;

@end
```
上面的这个代码

1. 我们定义了一个界面最终需要的数据的 property，这里是 `openSubjects`，这个 property 会存储用户打开的科目列表，他的类型是`APESubject`。
2. 我们还会定义一个接口来请求 openSubject 数据。

DataController 这一层是一个灵活性很高的部件，一个 DataController 可以复用更小的 DataController，这一类更小的 DataController 通常只会包含纯粹的或是更抽象的 Model 相关的逻辑，例如网络请求，数据库请求，或是数据加工等。我们称这一类 DataController 为 Model Related Data Controller。

Model Related Data Controller 通常会为上层提供正交的数据：

```
// APEHomePracticeDataController.m
@interface APEHomePracticeDataController ()

@property (nonatomic, strong, nonnull) APESubjectDataController *subjectDataController;

@end

@implementation APEHomePracticeDataController

- (void)requestSubjectDataWithCallback:(nonnull APECompletionCallback)callback {
    APEDataCallback dataCallback = ^(NSError *error, id data) {
        callback(error);
    };
    [self.subjectDataController requestAllSubjectsWithCallback:dataCallback];
    [self.subjectDataController requestUserSubjectsWithCallback:dataCallback];
}

- (nonnull NSArray<APESubject *> *)openSubjects {
    return self.subjectDataController.openSubjectsWithCurrentPhase ?: @[];
}

@end
```

在我们的 `APEHomePraticeDataController` 的实现中，就包含了一个 `APESubjectDataController`，这个 `subjectDataController` 会负责请求 All Subjects 和 User Subjects，并将其加工成上层所最终需要的 Open Subjects。（备注：这个例子里面的 callback 会回调多次是猿题库产品的需求，如有需要，可在这一层控制请求都完成后再调用上层回调）

事实上，Model Related Data Controller 可以一般性的认为就是大家经常在写的 Model 层代码，例如 UserAgent，UserService，PostService 之类的服务。之后读者若想重构就项目成这个架构，大可以不必纠结于形式，直接在 DataController 里调用旧有代码的逻辑即可，如图下面这样的行为都是允许的：

<img src="{{ size.url }}/assets/post/arch2.png" alt="Drawing" style="width: 640px;"/>

### ViewModel

每一个 View 都会有一个对应的 ViewModel，这个 ViewModel 会包含展示这个 View 所需要的所有数据。

我们会使用工厂方法来创建 View Model，例如这个例子里，Subject View Model 不需要关心传递给他是什么样的 Subject，所有的课目或者只是用户开启的科目。


```
@interface APEHomePracticeSubjectsViewModel : NSObject

@property (nonatomic, strong, nonnull) NSArray<APEHomePracticeSubjectsCollectionCellViewModel *>
*cellViewModels;

@property (nonatomic, strong, nonnull) UIColor *backgroundColor;

+ (nonnull APEHomePracticeSubjectsViewModel *)viewModelWithSubjects:(nonnull NSArray<APESubject *>
 *)subjects;

@end
```
ViewModel 可以包含更小的 ViewModel，就像 View 可以有 SubView 一样。SubjectView 的内部是由一个`UICollectionView`实现的，所以我们也给了对应的 Cell 设计了一个 ViewModel。

需要额外注意的是，ViewModel 一般来说会包含的显示界面所需要的所有元素，但粒度是可以控制。一般来说，我们只把会因为业务变化而变化的部分设为 ViewModel 的一部分，例如这里的 titleColor 和 backgroundColor 会因为主题不同而变化，但字体的大小（titleFont）却是不会变的，所以不需要事无巨细的都加到 ViewModel 里。

```
@interface APEHomePracticeSubjectsCollectionCellViewModel : NSObject

@property (nonatomic, strong, nonnull) UIImage *image;
@property (nonatomic, strong, nonnull) UIImage *highlightedImage;
@property (nonatomic, strong, nonnull) NSString *title;
@property (nonatomic, strong, nonnull) UIColor *titleColor;
@property (nonatomic, strong, nonnull) UIColor *backgroundColor;

+ (nonnull APEHomePracticeSubjectsCollectionCellViewModel *)viewModelWithSubject:(nonnull
APESubject *)subject;
+ (nonnull APEHomePracticeSubjectsCollectionCellViewModel *)viewModelForMore;

@end

```
### View
View 只需要定义好装配 ViewModel 的接口和定义好 UI 回调事件即可：

```
@protocol APEHomePracticeSubjectsViewDelegate <NSObject>

- (void)homePracticeSubjectsView:(nonnull APEHomePracticeSubjectsView *)subjectView
             didPressItemAtIndex:(NSInteger)index;

@end

@interface APEHomePracticeSubjectsView : UIView

@property (nonatomic, strong, nullable, readonly) APEHomePracticeSubjectsViewModel *viewModel;
@property (nonatomic, weak, nullable) id<APEHomePracticeSubjectsViewDelegate> delegate;

- (void)bindDataWithViewModel:(nonnull APEHomePracticeSubjectsViewModel *)viewModel;

@end
```
渲染界面的时候，完全依靠 ViewModel 进行，包括 View 的 SubView 也会使用 ViewModel 里面的子 ViewModel 渲染。

```
- (void)bindDataWithViewModel:(nonnull APEHomePracticeSubjectsViewModel *)viewModel {
    self.viewModel = viewModel;
    self.backgroundColor = viewModel.backgroundColor;
    [self.collectionView reloadData];
    [self setNeedsUpdateConstraints];
}
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:
(NSIndexPath *)indexPath {
    APEHomePracticeSubjectsCollectionViewCell *cell = [collectionView
dequeueReusableCellWithReuseIdentifier:@"Cell" forIndexPath:indexPath];
    if (0 <= indexPath.row && indexPath.row < self.viewModel.cellViewModels.count) {
        APEHomePracticeSubjectsCollectionCellViewModel *vm =
self.viewModel.cellViewModels[indexPath.row];
        [cell bindDataWithViewModel:vm];
}
    return cell;
}
```

至此，我们就完成了所有的步骤。我们回过头再看一下 ViewController 的职责就回变的非常简单，装配好 View，向 DataController 请求数据，装配 ViewModel，配置给 View，接收 View 的UI事，一切复杂的操作都能够的代理出去。

## 总结
### 优点
通过上面的例子我们可以看到，这个架构有几个优点：

**层次清晰，职责明确**：和界面有关的逻辑完全划到 ViewModel 和 View 一遍，其中 ViewModel 负责界面相关逻辑，View 负责绘制；Data Controller 负责页面相关的数据逻辑，而 Model 还是负责纯粹的数据层逻辑。 ViewController 仅仅只是充当简单的胶水作用。

**耦合度低，测试性高**：除开 ViewController 外，各个部件可以说是完全解耦合的，各个部分也是可以完全独立测试的。同一个功能，可以分别由不同的开发人员分别进行开发界面和逻辑，只需要确立好接口即可。

**复用性高**：解耦合带来的额外好处就是复用性高，例如同一个View，只需要多一个工厂方法生成 ViewModel，就可以直接复用。数据逻辑代码不放在 ViewController 层也可以更方便的复用。

**学习成本低**: 本质上来说，这个架构属于对 MVC 的优化，主要在于解决 Massive View Controller 问题，把原本属于 View Controller 的职责根据界面和逻辑部分相应的拆到 ViewModel 和 DataController 当中，所以是一个非常易于理解的架构设计，即使是新手也可以很快上手。

**开发成本低**: 完全不需要引入任何第三方库就可以进行开发，也避免了因为 MVVM 维护成本高的问题。

**实施性高，重构成本低**：可以在 MVC 架构上逐步重构的架构，不需要整体重写，是一种和 MVC 兼容的设计。

###缺点
不可否认的是，这个设计也有其相应的缺点，由于其把传统 MVVM 里面的 VM 拆成两部分，会照成下面的一些情况：

1. 当页面的交互逻辑非常多时，需要频繁的在 DC-VC-VM 里来回传递信息，造成了大量胶水代码。
2. 另外，由于在传统的 MVVM 中 VM 原本是一体的，一些复杂的交互本来可以在 VM 中直接完成测试，如今却需要同时使用 DC 和 VM 并附上一些胶水代码才能进行测试。
3. 没有了 Binding，代码写起来会更费劲一点（仁者见仁，智者见智）。

## 后记
MVVM 是一个很棒的架构，私底下我也会用其来做一些个人项目，但在公司项目里，我会更慎重的考虑个中利弊。我做这个设计的时候，心仪 MVVM 的种种好处，又忌惮于它的种种坏处，再考虑到团队的开发和维护成本，所以最终设计成了如今这样。

个人认为，好的架构设计的都是和团队以及业务场景息息相关的。我们这套架构帮助我们解决了 ViewController 代码堆积的问题，也带来了更清晰明了的代码层级和模块职责，同时没有引入过多的复杂性。希望大家也能充分理解这套架构的适用场景，在自己的 APP 架构设计中有所借鉴。

Lancy

2015.12.30