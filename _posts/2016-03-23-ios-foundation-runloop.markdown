---
layout:     post
title:      "RunLoop剖析"
subtitle:   "RunLoop，基础，归纳笔记"
date:       2016-02-24
author:     "Alla"
header-img: "img/post-bg-js-module.jpg"
tags:
- iOS
- RunLoop
---


## Foreword

>RunLoop是iOS系统运行的核心与基础
RunLoop，顾名思义，就是运行着的循环。它为让线程更高效的工作而生，用来监听循环中的事件，调度线程工作，并在没有接收到
事件时，让线程进入睡眠状态，以减少系统不必要的性能开销。

---

## Catalog

1.  [相关概念](#相关概念)
    1. [定义](#定义)
    2.  [结构](#结构)
        1. [Mode](#Mode)
        2. [ModeItem](#ModeItem)
2.  [生命周期](#生命周期)
3.  [运行机制](#运行机制)
    1. [RunLoop Observers](#RunLoop&nbsp&nbspObservers)
    2. [RunLoop事件队列](#RunLoop事件队列)
    3. [何时使用RunLoop](#何时使用RunLoop)
    4. [RunLoop对象的线程安全问题](#Runloop对象的线程安全问题)
4. [应用](#应用)
    1. [自动释放池](#自动释放)
    2. [事件监听](#事件监听)
    3. [定时器](#定时器)


## 相关概念

##### 定义 

通常，一个线程一次只能执行一个任务，当任务结束后就会被销毁。如果需要一个机制，来让线程能随时处理事件但并不退出，通常代码逻辑如下：

```
function eventLoop() {
    doInit();
    do {
        //睡眠，等待被唤醒
        var waker ＝ sleepforwakingup();
        var message = getnextmessage();
        processmessage(message);
    } while (message != quit);
}

```
这其实是一种设计模型，叫事件驱动模型，目前大部分的UI编程都是事件驱动模型，例如Android中的Looper和Handler，Windows中的消息循环，Javascript中的事件
处理，OS X/iOS中的RunLoop等。该模型的核心在于事件/消息管理机制的实现。

RunLoop就是这样一套提供了事件/消息管理能力并提供入口函数来执行eventLoop逻辑的对象。执行该函数后，当前线程就会一直处于该函数内部“接收消息——>睡眠，等待被唤醒-->处理”的循环中，
直至该循环结束，函数返回。

RunLoop的管理并不是完全自动的。我们仍然需要设计在适合的时机来启动RunLoop并正确响应输入事件。Cocoa和Core Foundation框架都提供了run loop objects来帮助配置和管理线程的RunLoop。
它们分别是CFRunLoopRef和NSRunLoop。CFRunLoopRef提供了C函数层级的API，而NSRunLoop是基于前者的封装，提供了面向对象的API。

需要注意的是，CFRunLoopRef相关的API是线程安全的，而NSRunLoop相关的并不是。我们的应用程序不需要显示的创建这些对象(run loop objects)。每个线程，包括程序的主线程都有与之对应的
run loop object。只有辅助线程才需要显示的运行它的RunLoop。在Carbon和Cocoa程序中，主线程会自动创建并运行对应的RunLoop来作为应用程序启动过程的一部分。

值得一提的是，CFRunLoopRef的代码是开源的，你可以在[这里]( http://opensource.apple.com/tarballs/CF/CF-855.17.tar.gz)找到整个Core Foundation的源码来进行研究。

##### 结构

RunLoop结构图

<img src = "http://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_0.png"  width="431" height="332"/>

一个RunLoop包含若干个Mode，每个Mode又包含若干个ModeItem。每次调用RunLoop的主函数时，只能指定其中一个Mode，这个Mode被称作CurrentMode。如果需要切换Mode，只能退出RunLoop

###### Mode

根据事件源的不同，RunLoop也被分成了许多种不同的模式，也就是被Cocoa和Core Foundation都封装了的RunLoop Mode。

RunLoop Mode可以理解为一个集合中包含所有件事的事件源和要通知的RunLoop中注册的观察者。每一次运行自己的RunLoop时，都需要显示或者隐示的指定其运行于哪一种Mode。在设置RunLoop
Mode后，你的RunLoop会自动过滤和其他Mode相关的事件源，而只监视和当前设置Mode相关的源(通知相关的观察者)。大多数时候，RunLoop都是运行在系统定义的默认模式上。

RunLoop Mode区分基于事件的源，而不是事件的种类。比如说你不能通过RunLoop Mode去只选择鼠标点击事件或者键盘输入事件。你可以使用RunLoop Mode去监听端口，暂停计时器或者改变其他源。

我们可以给Mode指定任意名称，但是它对应的集合内容不能是任意的。我们需要添加Input source，Timer source或者Observer到自定义的Mode。


系统已经定义的RunLoop Modes主要有以下几种：

*   NSDefaultRunLoopMode:大多数工作中默认的运行方式。
*   NSConnectionReplyMode:使用这个Mode去监听NSConnection对象的状态。
*   NSModalPanelRunLoopMode:是用这个Mode在Model Pannel情况下去区分事件（OSX开发中会遇到）。
*   NSEventTrackingRunLoopMode：使用这个Mode去跟踪来自用户交互的事件（比如UITableView上下滑动）。
*   NSRunLoopCommonModes：这是一个伪模式，其为一组run loop mode的集合。如果将input source加入此模式，意味着关联input source到Common Modes中包含的所有模式下。在iOS系统中,
NSRunLoopCommonModes包含NSDefaultRunLoopMode，NSTaskDeathCheckMode，NSEventTrackingRunLoopMode。通常情况下，我们可使用CFRunLoopAddCommonMode方法向Common Modes中添加自定义mode。

RunLoop运行时只能以一种固定的Mode运行，只会监控这个Mode下添加的Timer source和Input source。如果这个Mode下没有添加事件源，RunLoop会立刻返回。

RunLoop不能运行咋NSRunLoopCommonModes，因为NSRunLoopCommonModes是个Mode集合，而不是一个具体的Mode。我们可以在添加事件源的时候使用NSRunLoopCommonModes，只要RunLoop运行在NSRunLoopCommonModes中
任何一个Mode，这个事件源都可以被触发。

###### ModeItem
RunLoop从两种不同的事件源中来接收消息：

*  Input Source
*  Timer Source

###### Input Source

Input Source，也就是输入源。用来传递异步消息，通常消息来自另外的线程或者程序。

Input Source 有两个不同的种类：Port-Based Sources 和Custom Input Source。RunLoop本身并不关心Input Source是哪一种类型。系统会实现两种不同的Input Source供我们使用。
这两种不同类型的Input Source的区别在于：Port-Based Sources由内核自动发送，Custome Input Sources需要从其他线程手动发送。

####### Port-Based Sources
通过内置的端口相关的对象和函数，配置基于端口的Input Source。（比如在主线程创建子线程时传入一个NSPort对象，主线程和子线程可以进行通讯。NSPort对象会负责自己创建和配置Input Source。）

####### Custome Input Sources

我们可以使用Core Foundation里面的CFRunLoopSourceRef类型相关的函数来创建custome input source。

Cocoa框架为我们定义了一些Custome Input Sources，允许我们在线程中执行一系列selector方法：

```
//在主线程的RunLoop下执行指定的@selector 方法
performSelectorOnMainThread：withObject：waitUntilDone：
performSelectorOnMainThread：withObject：waitUntilDone：modes：

//在当前线程的RunLoop下执行指定的 @selector 方法
performSelector:onThread:withObject:waitUntilDone:
performSelector:onThread:withObject:waitUntilDone:modes:

//在当前线程的RunLoop延迟加载指定的@selector 方法
performSelector:withObject:afterDelay:
performSelector:withObject:afterDelay:inModes:

//取消当前线程的调用
cancelPreviousPerformRequestWithTarget:
cancelPreviousPerformRequestWithTarget:selector:object:

```

和Port-Based Sources一样，这些selector的请求会在目标线程中序列化，以减缓线程中多个方法执行带来的同步问题。

和Port-Based Sources不一样的是，一个selector执行完之后会自动从当前RunLoop中移除。

###### Timer Sources

Timer Source在预设的时间点同步的传递消息。Timer是线程同志自己做某件事的一种方式。

Foundation中NSTimer Class提供了相关方法来设置Timer Source。需要注意的是除了schedeledTimerWithTimeInterval开头的方法创建的Timer都需要手动添加到当前的RunLoop中。
scheduledTimerWithTimeInterval创建的timer会自动以Default Mode加载到当前RunLoop中。

Timer在选择使用一次后，在执行完成时，会从RunLoop中移除。选择循环时，会一直保存在当前RunLoop中，直到调用invalidate方法。

关于如何配置RunLoop Source ，推荐大家看这篇[文章](http://chun.tips/blog/2014/10/20/zou-jin-run-loopde-shi-jie-er-:ru-he-pei-zhi-run-loop-sources/)


##### 生命周期

上文已经提到RunLoop是为线程而生，所以，RunLoop的生命周期也是与线程的生命周期紧密相连的。

容器类视图控制器包括一个或多个子内容类视图控制器，一般用来负责不同场景下，子视图控制器之间的切换和展现。关于容器类试图控制器会在第二期讲到，这里就不赘述了。

内容类视图控制器管理着App中各自对应界面的视图，是我们在开发过程中使用频率极高的视图控制器类型。


## 运行机制

#### RunLoop Observers

Cocoa框架中很多机制比如CAAnimation等都是由RunLoopObserver触发的。observer到当前状态的变化进行通知。


```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
kCFRunLoopEntry = (1UL << 0),  //即将进入RunLoop
kCFRunLoopBeforeTimers = (1UL << 1),//即将处理Timer
kCFRunLoopBeforeSources = (1UL << 2),//即将处理Source
kCFRunLoopBeforeWaiting = (1UL << 5),//即将进入休眠
kCFRunLoopAfterWaiting = (1UL << 6),//刚从睡眠中唤醒
kCFRunLoopExit = (1UL << 7),//即将退出RunLoop
kCFRunLoopAllActivities = 0x0FFFFFFFU//
};

```


事件源是同步或者异步的事件驱动时触发，而Run Loop Observer则在Run Loop本身进入某个状态时得到通知:

* Run Loop 进入的时候
* Run Loop 处理一个Timer的时候
* Run Loop 处理一个Input Source的时候
* Run Loop 进入睡眠的时候
* Run Loop 被唤醒的时候，在唤醒它的事件被处理之前
* Run Loop 停止的时候

Observer需要使用Core Foundataion框架。和Timer一样，Run Loop Observers也可以使用一次或者选择repeat。如果只使用一次，Observer会在它被执行后
自己从Run Loop中移除。而循环的Observer会一直保存在Run Loop中。 

#### RunLoop事件队列

Run Loop本质是一个处理事件源的循环。我们对Run Loop的运行时具有控制权，如果当前没有时间发生，Run Loop会让当前线程进入睡眠模式，来减轻CPU压力。如果有事件发生，Run Loop就处理事件并通知相关的Observer。具体的顺序如下:

1) Run Loop进入的时候，会通知Observer

2) Timer即将被触发时，会通知Observer

3) 有其它非Port-Based Input Source即将被触发时，会通知Observer

4) 启动非Port-Based Input Source的事件源

5) 如果基于Port的Input Source事件源即将触发时，立即处理该事件，并跳转到9

6) 通知Observer当前线程进入睡眠状态

7) 将线程置入睡眠状态直到有以下事件发生：

1. Port-Based Input Source被触发。

2. Timer被触发。 

3. Run Loop设置的时间已经超时。 

4. Run Loop被显示唤醒。

8) 通知Observer线程将要被唤醒

9) 处理被触发的事件：

1. 如果是用户自定义的Timer，处理Timer事件后重启Run Loop并直接进入步骤2。

2. 如果线程被显示唤醒又没有超时，那么进入步骤2。 

3. 如果是其他Input Source事件源有事件发生，直接传递这个消息。

10) 通知Observer Run Loop结束，Run Loop退出。

基于非Timer的Input source事件被处理后，Run Loop在将要退出时发送通知。基于Timer source处理事件后不会退出Run Loop。

#### 何时使用 RunLoop

我们应该只在创建辅助线程的时候，才显示的运行一个Run Loop。对于辅助线程，我们仍然需要判断是否需要启动Run Loop。比如我们使用一个线程去处理
一个预先定义的长时间的任务，我们应当避免启动Run Loop。下面是官方Document提供的使用Run Loop的几个场景:

* 需要使用Port-Based Input Source或者Custom Input Source和其他线程通讯时
* 需要在线程中使用Timer
* 需要在线程中使用上文中提到的selector相关方法
* 需要让线程执行周期性的工作

#### RunLoop对象的线程安全问题

使用Core Foundation中的方法通常是线程安全的，可以被任意线程调用。如果修改了Run Loop的配置然后需要执行某些操作，我们最好是在Run Loop所在的
线程中执行这些操作。

使用Foundation中的NSRunLoop类来修改自己的Run Loop，我们必须在Run Loop的所在线程中完成这些操作。在其他线程中给Run Loop添加事件源或者Timer
会导致程序崩溃。


##应用
#### 自动释放池
App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

#### 事件响应
苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考这里。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。


#### 手势识别
当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。

当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

#### 界面更新
当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

这个函数内部的调用栈大概是这样的：

```
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()
    QuartzCore:CA::Transaction::observer_callback:
        CA::Transaction::commit();
            CA::Context::commit_transaction();
            CA::Layer::layout_and_display_if_needed();
                CA::Layer::layout_if_needed();
                [CALayer layoutSublayers];
                    [UIView layoutSubviews];
                        CA::Layer::display_if_needed();
                            [CALayer display];
                                [UIView drawRect];
```

#### 定时器
NSTimer 其实就是 CFRunLoopTimerRef，他们之间是 toll-free bridged 的。一个 NSTimer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。
例如 10:00, 10:10, 10:20 这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。

如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。就比如等公交，如果 10:10 时我忙着玩手机错过了那个点的公交，那我只能等 10:20 这一趟了。

CADisplayLink 是一个和屏幕刷新率一致的定时器（但实际实现原理更复杂，和 NSTimer 并不一样，其内部实际是操作了一个 Source）。
如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 NSTimer 相似），造成界面卡顿的感觉。
在快速滑动TableView时，即使一帧的卡顿也会让用户有所察觉。Facebook 开源的 AsyncDisplayLink 就是为了解决界面卡顿的问题，
其内部也用到了 RunLoop。

#### PerformSelecter
当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。
所以如果当前线程没有 RunLoop，则这个方法会失效。

当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。


#### 关于GCD
实际上 RunLoop 底层也会用到 GCD 的东西，比如 RunLoop 是用 dispatch_source_t 实现的 Timer（评论中有人提醒，NSTimer 是用了 
XNU 内核的 mk_timer，我也仔细调试了一下，发现 NSTimer 确实是由 mk_timer 驱动，而非 GCD 驱动的）。
但同时 GCD 提供的某些接口也用到了 RunLoop， 例如 dispatch_async()。

当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，
并从消息中取得这个 block，并在回调 __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__() 里执行这个 block。
但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

#### 关于网络请求
iOS 中，关于网络请求的接口自下至上有如下几层:

1.  CFSocket
2.  CFNetwork       ->ASIHttpRequest
3.  NSURLConnection ->AFNetworking
4.  NSURLSession    ->AFNetworking2, Alamofire

• CFSocket 是最底层的接口，只负责 socket 通信。

• CFNetwork 是基于 CFSocket 等接口的上层封装，ASIHttpRequest 工作于这一层。

• NSURLConnection 是基于 CFNetwork 的更高层的封装，提供面向对象的接口，AFNetworking 工作于这一层。

• NSURLSession 是 iOS7 中新增的接口，表面上是和 NSURLConnection 并列的，但底层仍然用到了 NSURLConnection 的部分功能 (比如 com.apple.NSURLConnectionLoader 线程)，AFNetworking2 和 Alamofire 工作于这一层。

下面主要介绍下 NSURLConnection 的工作过程。

通常使用 NSURLConnection 时，你会传入一个 Delegate，当调用了 [connection start] 后，这个 Delegate 就会不停收到事件回调。
实际上，start 这个函数的内部会会获取 CurrentRunLoop，然后在其中的 DefaultMode 添加了4个 Source0 (即需要手动触发的Source)。
CFMultiplexerSource 是负责各种 Delegate 回调的，CFHTTPCookieStorage 是处理各种 Cookie 的。

当开始网络传输时，我们可以看到 NSURLConnection 创建了两个新线程：com.apple.NSURLConnectionLoader 和 com.apple.CFSocket.private。
其中 CFSocket 线程是处理底层 socket 连接的。NSURLConnectionLoader 这个线程内部会使用 RunLoop 来接收底层 socket 的事件，
并通过之前添加的 Source0 通知到上层的 Delegate。

<img src = "http://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_network.png"  width="452" height="347"/>

NSURLConnectionLoader 中的 RunLoop 通过一些基于 mach port 的 Source 接收来自底层 CFSocket 的通知。
当收到通知后，其会在合适的时机向 CFMultiplexerSource 等 Source0 发送通知，同时唤醒 Delegate 线程的 RunLoop 来让其处理这些通知。
CFMultiplexerSource 会在 Delegate 线程的 RunLoop 对 Delegate 执行实际的回调。

