---
layout:     post
title:      "谈谈我对UIViewController的理解（一）"
subtitle:   "UIKit，UIViewController，归纳笔记"
date:       2016-02-23
author:     "Alla"
header-img: "img/post-bg-ios-uikit-uiviewcontroller.jpg"
tags:
- iOS
- UIKit
- UIViewController
---



## Foreword

> UIViewController，UI开发的基石!

对于iOS开发者，UIViewController并不陌生，它是UIKit框架为我们提供的用来管理App视图的一个基础机制类，我们可以用它来管理视图，处理用户事件以及不同UIViewController之间的交互。总而言之，所有与视图相关的操作逻辑最好都只在该类实现。

App中至少有一个UIViewController，每一个UIViewController对应一个视图。每个UIViewController均为我们提供了一个根视图，并且一般需要添加视图，作为其子视图来达到自定义界面的目的。

UIViewController已经为我们提供了视图内存管理，展示，屏幕旋转等一系列方法回掉，我们只需根据需求合理的在相关方法中来进行相关逻辑处理即可。

---

## Catalog

1. [分类](#分类)
2. [功能](#功能)
3. [管理](#管理)
    1. [内存](#内存)
    2. [展示](#展示)
    3. [事件接收](#事件接收)
4. [规范](#规范)

---

## 分类
从功能上来划分，UIViewController分为两类：


*   容器类视图控制器
*   内容类视图控制器


容器类视图控制器包括一个或多个子内容类视图控制器，一般用来负责不同场景下，子视图控制器之间的切换和展现。关于容器类试图控制器会在第二期讲到，这里就不赘述了。

内容类视图控制器管理着App中各自对应界面的视图，是我们在开发过程中使用频率极高的视图控制器类型。


## 功能
从命名，不难看出，UIViewController是用来管理视图的控制器。没错，控制器的核心功能就是管理视图。


## 管理

UIViewController从以下几个方面来管理其负责的视图

*   内存
*   展示
*   事件接收

### 内存

UIViewController也和其他的Objective-C对象一样，有创建，初始化，销毁等过程。另外，还提供加载视图，视图已加载完毕，内存不足等方法回调。如下：

```
//初始化
- (instancetype)init;
- (instancetype)initWithNibName:(nullable NSString *)nibNameOrNil bundle:(nullable NSBundle *)nibBundleOrNil
//开始加载视图
- (void)loadView;
//视图加载完毕
- (void)viewDidLoad;
//视图即将被回收和已经被回收
- (void)viewWillUnLoad;
- (void)viewDidUnLoad;
//收到内存警告
- (void)didReceiveMemoryWarning;

```

##### 初始化与开始加载视图

在iOS开发中，视图的生成与样式布局编辑有两种实现方式，一种是用Storyboard/Xib，另外一种则是用代码。后者的优先级高于前者。

无论用哪种方式来实现，UIViewController的初始化都可以用init方法来实现。

当用init方法初始化UIViewController时，系统会先调用initWithNibName方法，如果该方法中的nibNameOrNil参数为nil时，loadView方法会尝试加载名字和UIViewController类名相同的xib文件，若该文件不存在，我们需要手动调用[self setView:(UIView *)view]方法或者重写loadView方法。从这可以看出，加载xib文件的重任是initWithNib方法和loadView方法共同来承担的。



以下是苹果官方文档关于initWithNib方法的说明：


>The designated initializer. If you subclass UIViewController, you must call the super implementation of this method, even if you aren't using a NIB.  (As a convenience, the default init method will do this for you,and specify nil for both of this methods arguments.) In the specified NIB, the File's Owner proxy should have its class set to your view controller subclass, with the view outlet connected to the main view. If you invoke this method with a nil nib name, then this class' -loadView method will attempt to load a NIB whosename is the same as your view controller's class. If no such NIB in fact exists then you must either call -setView: before -view is invoked, or override the -loadView method to set up your views programatically.

也就是说如果有对应的xib文件，我们就不能调用[self setView:(UIView *)view]方法或者重写loadView方法了。

如果我们需要手工维护子views，就需要调用或重写上述方法了。

那么如何重写loadView方法呢？在讨论这个问题之前，我们需要首先了解一下loadView方法的默认实现：先寻找有关可用的xib文件的信息，根据这个信息来加载xib文件，如果没有有关xib文件的信息，默认会实现一个空白的UIView对象，然后让这个对象成为controller的主View。所以，重载这个函数时，我们也应该这样，把子类的view赋给view属性，并且重载的这个函数不应该调用super。

为什么不要调[super loadView]方法？因为系统首先会再次寻找对应的xib文件并且我们自定义的View最终会代替[super loadView]生成的View，这样做，多多少少会影响性能的。下面是一个重写的例子

```
- (void)loadView { 
UIView *view = [[UIView alloc] initWithFrame:[[UIScreen mainScreen] applicationFrame]; 
[view setBackgroundColor:_color] ; 
self.view = view;
}

```
综上所述，以下几点需要注意：

*   当调用self.view时，如果此时值为nil，就会调用loadView方法。

*   loadView方法执行完以后必定会调用viewDidLoad方法，如果此时self.view还是nil，根据第一条注意事项，系统就会调用loadView方法，这样就会引起死循环。所以，为了避免死循环，我们最迟要在viewDidLoad方法中结束self.view是nil的状态。

*   永远不要主动调用loadView方法。

##### 视图加载完毕

注意，这里的视图，指的是根视图。也就是说当根视图加载到内存后，才会回掉至该方法中。因为此时该视图还没在window中，所以此时根视图的bounds值还是不准确的。在该方法中，我们可以为根视图做一些额外的配置以及为其添加一些子视图。子视图往往被声明为UIViewController的属性，所以其初始化及配置我们最好采用苹果layloading的加载机制，在其getter和setter方法中来进行。

该方法一般只有在第一次进入视图时，会执行。

##### 内存不足
当系统因App占用过多内存时，UIViewController中的didReceiveMemoryWarning方法会被执行，在该方法中，我们需要把一些不重要的内存缓存清理掉，如果内存仍然吃紧，viewWillUnLoad和

和viewDidUnLoad方法将依次执行，根视图将从内存中清除，在viewDidUnLoad方法中，我们需要把UIViewController中我们声明的成员变量统统释放掉。



### 展示

视图的展示主要是以下方法

```
- (void)viewWillAppear; //视图将要被加载如window的视图层次中去，会被调用多次
- (void)viewDidAppear;
- (void)viewWillLayoutSubviews; //根视图将布局其子视图，我们一般在该方法中调整子视图的布局
- (void)viewDidLayoutSubviews;
- (void)viewWillDisappear;
- (void)viewDidDisappear;

```

### 事件接收

事件一般来自于用户的操作或者是设备自身的变化，主要包含并且不局限于以下几个方面：

*   屏幕旋转
*   delegate回调
*   target-action回调
*   notification回调
*   gesture回调

## 规范

通过以上对UIViewController的分析，个人觉得UIViewController的代码大概如下：

```
@property (nonatomic,strong) UILabel *titleLabel;
...

#pragma mark - life cycle
- (void)init;
- (void)viewDidLoad;
...
#pragma mark - user interactive
...
#pragma mark rotation
- (BOOL)shouldAutorotate;
...
#pragma mark notification callback
...
#pragma mark action
...
#pragma mark delegate
...
#pragma mark geture callback
...
#pragma mark - private methods
...
#pragma mark - getters and setters
...

```

## 完结
这次我们聊了对UIViewController的总体认识以及在开发中需要注意的地方，在该系列第二篇中，我会为大家解读容器类试图控制器。
