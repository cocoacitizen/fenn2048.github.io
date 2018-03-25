---
layout:     post
title:      " 浅谈iOS11适配（一）"
subtitle:   "UIKit，UIScrollView，归纳笔记"
date:       2017-08-27
author:     "Alla"
header-img: "img/post-bg-ios-uikit-uiviewcontroller.jpg"
tags:
- iOS11
- UIKit
- UIScrollView
---


今年6月份，苹果在WWDC上发布了iOS11系统。UI的风格进行了更新，更加简洁，动态化。在控制中心新增了屏幕录制，关闭蜂窝连接等功能。新推出的Core ML和ARKit框架令人眼前一亮。从iOS10.3到11.0的API更新较多，本系列将会介绍以下适配点：
* Safe Area
* TableView：Self-Sizing
* File Provider

##Safe Area

###背景
从iOS7开始，当视图控制器结构中有 navigation bar，tab bar或tool bar时，bar的属性的默认值为YES，且ViewController的高度变为整个屏幕的高度。当视图中有列表滑动时，可以透过具有高斯模糊的bar。

为了确保视图不被这些bars遮挡，我们可以在自动布局（Masonary）中使用topLayoutGuide和bottomLayoutGuide属性。

```
UIButton *bottomButton = [[UIButton alloc] init];
[self.view addSubview:bottomButton];
[bottomButton mas_makeConstraints:^(MASConstraintMaker *make) {
    make.height.equalTo(@40);
    make.left.and.right.equalTo(self.view);
    make.bottom.equalTo(self.mas_bottomLayoutGuide);
}];
```
计算方法：
* topLayoutGuide：ViewController中的View被顶部的bar（navigation bar）遮挡的高度。
* bottomLayoutGuide：ViewController中的View被底部的bar（tool bar）遮挡的高度。

如果使用手动布局，可通过上述两种guide的length属性值获取视图被bar遮挡的高度。

-------

###新特性

#### 概念
iOS11中，屏幕中视图被bars遮挡以外可见的区域，称为Safe Area。topLayoutGuide和bottomLayout被safeAreaLayoutGuide替代，我们可以用它创建constraints来进行自动布局。如果没有使用自动布局，我们可以通过safeAreaInsets来获取系统建议的该视图的内边距，如下图：

![](/img/15036689234419.png)

容器类视图控制器tab bar和navigation controller会通过safeAreaInsets和safeAreaLayoutGuide来指导我们调节其内容控制器中视图的Safe Area。上述两属性值为只读属性。我们还可用additionalSafeAreaInsets属性来改变Safe Area所表示的区域。

当视图真正被添加到视图结构中，safeAreaInsets和safeAreaLayoutGuide的参考值才会被正确设置。所以我们在UIViewController的viewDidAppear回调中来使用这两个属性值。
当Safe Area所表示区域变化（视图旋转等）时，我们可以通过UIViewController的viewSafeAreaInsetsDidChange和UIView的safeAreaInsetsDidChange回调方法来进行额外的处理。

-------
#### 使用

##### UIView
如果在UIViewController中，要添加自己的bars，增加safeAreaInsets来改变Safe Area区域大小，我们可通过addtionalSafeAreaInsets来改变。
##### UIScrollView
从iOS7开始，UIViewController中新增加了automaticallyAdjustsScrollViewInsets属性，来告诉其容器视图控制器是否需要根据status bar，search bar，navigation bar或 tab bar来调整根视图的直接子视图中第一个视图contentInset的边距，且该视图必须为UIScrollView或其子类。默认值为YES。

苹果这样的设计初衷是好的，避免了在某些情况下，需要手动设置UIScrollView的contentInset的麻烦。但是，某些情况下，是兼顾不到的，比如，在UIScrollView为根视图的间接子视图中，即使该属性设置为YES，边距仍不会自动调节。某些情况下，希望能自动调节根视图中指定UIScrollView的contentInset边距而不是第一个，这样的话，UIViewController的automaticallyAdjustsScrollViewInsets就不能满足我们的需求了。另外，有时候，我们手动设置contentInset和系统为我们自动调节contentInset的时机有冲突，最终设置的contentInset可能就不是我们想要的值。

为了满足开发者的这些需求，苹果在iOS11进行了走心的设计。
首先，UIScrollView新增加了只读属性adjustedContentInset，即调节后的内容边距。它才是UIScrollView最终呈现在屏幕中的内容边距。一般情况下，adjustedContentInset的值为conentInset和SafeAreaInsets之和，但是某些情况下，不考虑SafeAreaInsets，这完全取决于UIScrollView的另一个新增属性contentInsetAdjustmentBehavior，UIViewController的automaticallyAdjustsScrollViewInsets属性被打入冷宫，新属性为枚举类型，有以下四种选项供开发者选择：
* UIScrollViewConentInsetAdjustmentNever
系统不调整UIScrollView的adjustedContentInset，这个时候，adjustedContentInset的值完全取决于contentInset的值，和原来的automaticallyAdjustsScrollViewInsets为NO的效果大抵相同。
* UIScrollViewContentInsetAdjustmentScrollableAxes
当可滚动（contentSize.width/height > frame.size.width/height 或 alwaysBounceHorizontal/Vertical = YES）时，adjustedContentInset = safeAreaInsets+contentInset，否则走Never选项的计算逻辑。
* UIScrollViewContentInsetAdjustmentAutomatic
该类型为默认选择类型，也是最复杂的类型，通过下图列表，我们一探究竟：


-------

| 编号 | automaticallyAdjustsScrollViewInsets | 是否为第一个scrollview | 是否可滑动 | 系统是否会调节 |
| :-: | :-: | :-: | :-: | :-: |
| 1 | YES | YES | YES | YES |
| 2 | YES | YES | NO | YES |
| 3 | YES | NO | YES | YES |
| 4 | YES | NO | NO | NO |
| 5 | NO | YES | YES | NO |
| 6 | NO | NO | NO | NO |
| 7 | NO | YES | NO | NO |
| 8 | NO | NO | YES | YES |

通过上表，我们可以发现，影响系统自动调节的因素有三个：
1. UIViewController的automaticallyAdjustsScrollViewInsets
2. UIScrollView是否为根视图遍历到的第一个视图。注意，在这里，UIScrollView可以不是根视图的第一个视图，也可以是根视图纵向遍历子视图的第一个非直接子视图。
3. 是否可滑动，该点判断条件，参考ScrollableAxes选项。

通过上图中的对比，我们可以得出以下结论：
1. 只要automaticallyAdjustsScrollViewInsets值为YES，第一个scrollview总是会被自动调节。
2. 只要可滑动，非第一个scrollview总是会被调节。
* UIScrollViewContentInsetAdjustmentAlways
这个就不难理解了，任何情况下，系统总是会调整UIScrollView的adjustedContentInset。

在适配测试的过程中，发现UIWebView中UIScrollView的contentInsetAdjustmentBehavior总是UIScrollViewContentInsetAdjustmentAlways，还请大家注意哈。

适配建议：
更改UIScrollView的ContentInset的地方，将contentInsetAdjustmentBehavior选项值设置为Never，自己手动通过contentInset来进行内容边距的适配。

下一章，会给大家带来UITableView的Self-Sizing相关的特性介绍和适配，敬请期待。


