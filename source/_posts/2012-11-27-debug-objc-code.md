---
title: 从一段奇葩的objc代码看代码规范的重要性
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - debug
  - iOS
  - OBJC
  - tableview
---
## 背景介绍
昨天，在和我一个朋友讨论，到底是用`self.propertyName`还是`_propertyName`来访问property，我认为应该使用`self.propertyName`，因为我在听Stanford
Open
Course的时候，苹果的工程师告诫要使用`self.propertyName`，不要使用`_propertyName`。而朋友认为应该使用`_propertyName`，因为google
objc code style认为最好不要用`self.propertyName`。

我没看过google objc code style，我只看过objective c programming
guide。在我的理解里property的作用在于根据参数生成相应的getter和setter。`self.propertyName`本质上既是调用getter函数的，而`_propertyName`直接访问成员函数，因为相应参数生成的getter和setter是不会被调用的。

再说，我还是决定相信apple，而不是google，毕竟Objc还是apple在支持和维护。

## 上代码
重点来了，朋友为了说服我`self.property`是有问题的，发了一段代码过来，这段代码非常奇葩，可以点[这里](http://lancy.applesysu.com/wp-content/uploads/2012/11/FuckTableView.zip)下载，或者直接看代码，代码不算很长，简单的说，是要实现一个功能，一个tableview右上角有一个刷新按钮，每次刷新会改变dataArray（setupData），然后刷新tableview。

```objective-c
#import "ViewController.h"

@interface ViewController () <UITableViewDataSource, UITableViewDelegate>

@property (strong, nonatomic) UITableView *tableView;
@property (strong, nonatomic) NSArray *dataArray;
@property (assign, nonatomic) BOOL flag;

@end
    
@implementation ViewController

- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self)
    {
        //按这个按钮本来是tableview会变化的，但是现在调用了reloadData之后，不会调用cellForRowAtIndexPath这个方法。
        UIBarButtonItem *rightItem = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemEdit target:self action:@selector(add:)];
        self.navigationItem.rightBarButtonItem = rightItem;
        
        //设置数据源
        [self setupData];
    }
    return self;
}

- (void)viewDidLoad
{
    [super viewDidLoad];
    [self.view addSubview:self.tableView];
}

#pragma mark - getter & setter

- (UITableView *)tableView
{
    if (_tableView == nil)
    {
        _tableView = [[UITableView alloc] initWithFrame:self.view.bounds
                                                  style:UITableViewStyleGrouped];
        _tableView.autoresizingMask = UIViewAutoresizingFlexibleHeight;
        _tableView.delegate = self;
        _tableView.dataSource = self;    
    }
    return _tableView;
}

#pragma mark - UITableView Delegate & Datasource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    return self.dataArray.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    static NSString *identifier = @"settingcell";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:identifier];
    if (cell == nil)
    {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:identifier];
    }
    
    cell.textLabel.text = [self.dataArray objectAtIndex:[indexPath row]];
    
    return cell;
}

- (void)setupData
{
    if (self.flag)
        self.dataArray = [NSArray arrayWithObjects:@"1", @"2", @"3", @"fuck", nil];
    else
        self.dataArray = [NSArray arrayWithObjects:@"1", @"2", @"3", nil];
    [self.tableView reloadData];
    
}

- (void)add:(id)sender
{
    self.flag = !self.flag;
    [self setupData];
}

@end
```

### 问题
这段代码是无效的，按下按钮之后，`setupData`被调用了，已经log确定`dataArray`已经改变，`tableview`的`delegate`和`datasource`都设置正确，确定`numberOfRowsInSection`被调用，奇葩的是`cellForRowAtIndexPath`没有调用，故而`tableview`没有改变。
#### 奇葩的来了
朋友跟我说，你只要把`[self.tableview reloadData]`改成`[_tableview
reloadData]`，他就生效了。是的，他就生效了。你设一个断点在这个地方，然后把`self.tableview`和`_tableview`po出来，发现他们的指针是一样的。朋友说写这个代码的那货折腾了一天，百思不得其解，最后得出结论`self.propertyName`就是坑爹。

## 生效的修改方法
朋友提供的：

1. 前面说的讲把`[self.tableview reloadData]`改成`[_tableview reloadData]`
2.
把`tableview`的getter函数的`init`里面的`self.view.bounds`改成`CGReckMake(0,0,
320, 480)`

朋友试图用这个两个方法来说明，`self.property`是坑爹的。

我在初步debug的时候，由于我是property的拥护者，property自动生成setter和getter函数，我是不支持重写getter函数的，所以我将getter函数删掉，把初始化代码移到`viewdidload`里面。然后代码就生效了。

但是即使代码生效了，还是没有找到问题的关键，仍然没办法解释为什么`[self.tableview
reloadData]`改成`[_tableview reloadData]`就能运行了，因为po出来的指针是完全一样的，这不科学。
## 真正的问题所在
在各种Stackoverflow，google无果之后，我还是着手准备深入debug。

通过各种断点和gdb，最后打印函数调用栈才让我发现了真正的问题所在。

整个程序的执行顺序是这样的：

1. `initWithNibName`（执行到`[self setupData]`，没执行完） -> 
2. 第一次setupData(执行到`[self.tableView reloadData]`，没执行完) -> 
3. 第一次执行tableview getter（到init，调用`self.view`，没执行完）->
4. `viewDidLoad`(到`addSubview:self.tableView`, 没执行完) ->
5. 第二次执行tableview getter（问题在这里！第一次执行的时候没有init玩，所以又会执行一次！）-> 
6. 回到4.viewDidLoad，这是add的subview是第二次的init而先init完的tableview ->
7. 回到3.第一次执行getter，（又alloc了一次tableView，这是`self.property`指向的是第一次init而后init完成的tableview））

所以，显示在界面上的tableview根本不是`self.tableview`指向的tableview，故而根本没法刷新（`cellForRowAtIndexPath`，是当需要显示的时候才会调用的）。

那为什么把`[self.tableview reloadData]`改成`[_tableview
reloadData]`就能生效了呢？因为这样在`initWithNibName`的第一次调用setupData，就不会在reload的时候调用tableview
getter，也就不会有后面一连串的连锁反应。之后顺利在`viewdidload`的时候只调用一次，完成init。

知道了问题的关键，还能有各种各样让他生效的方法，就不吐槽了。
## 正确的写法
这段奇葩代码带给我最大的感触就是，不好好写规范的代码，各种问题都会坑死你。我认为规范的写法应该是

1. 不要重写getter和setter函数，使用property生成的getter和setter
2. 不要在vc的init的函数里面初始化，尤其是初始化视图。而应该在viewdidload里面初始化，保证self.view已经生成。
3. 应该使用自顶向下的程序设计方法，保证程序的顺序执行和层次关系。不应该出现如上程序的跳来跳去的调用。

## 后记
帮人debug这种事情真心蛋疼，看不规范的代码像噩梦。

P.S.好想看objc和cocoa源码。。

Lancy
