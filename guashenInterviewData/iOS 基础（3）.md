[TOC] 

# iOS 面试基础题目

## 21. 当 Timer 以 `+ scheduledTimerWithTimeInterval` 的方式触发，当回调中定时改变 cell 中的视图，在滑动过程中发现 Timer 会暂停调用的原因是什么？如何解决？

RunLoop 只能运行在一种 Mode 下，如果要更换 Mode，当前的 Loop 也需要停下重启新的。根据 RunLoop 这个机制，ScrollView 在滑动过程中，`NSDefaultRunLoopMode` 的 Mode 或切换到 `UITrackingRunLoopMode` 来保证 ScrollView 的流畅滑动；此时会停止 `NSDefaultRunLoopMode` 的运行，从而 `NSTimer` 会停止触发。

解决方案：
* 第一种是更换 `NSTimer` 的 Mode

```Objective-C
[[NSRunLoop currentRunLoop] addTimer: self.timer forMode: NSRunLoopCommonModes];
```

* 或者开启一个新的线程，让 Timer 在新的线程中构造，这样 `NSTimer` 由子线程的 RunLoop 处理。

```Objective-C
[NSTread detachNewThreadSelector: @selector(timerFunction) toTarget: self withObject: nil];

- (void)timerFunction {
    self.timer = [NSTimer scheduledTimerWithTimerInterval: 1 target: self selector: @selector(updateTimer) userInfo: nil repeats: YES];
    [[NSRunLoop currentRunLoop] addTimer: self.timer forMode: NSDefaultRunLoopMode];
    // 非 main 线程 RunLoop 需要手动调用 run 方法
    [[NSRunLoop currentRunLoop] run];
}
```

## 22. RunLoop 和 RunLoopMode 的数据结构分别是怎样的？

推荐一下 ibireme 大佬的 [深入理解RunLoop
](https://blog.ibireme.com/2015/05/18/runloop/#base)。

数据结构给出 Core Foundation 中的定义：

```Objective-C
struct __CFRunLoopMode {
    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;    // Set
    CFMutableSetRef _sources1;    // Set
    CFMutableArrayRef _observers; // Array
    CFMutableArrayRef _timers;    // Array
    ...
};
 
struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set
    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set
    ...
};
```

记住一个 Mode 中，有“一个名字、两个源头、一个观察组和一个计时组”。一个 RunLoop 包含若干个 Mode，每个 Mode 又包含了若干个 Source/Timer/Observer。每次调用 RunLoop 主函数时，只能指定其中一个 Mode，这个 Mode 被称作 `CurrentMode`。如果需要切换 Mode，只能先退出当前的 Loop，再指定一个 Mode 重新进入，这样就可以分隔不同组的 Source/Timer/Observer，使其互不影响。

另外需要介绍这些关于 RunLoop 的引用：

* **CFRunLoopSourceRef**：是事件产生的地方。Source 分为 Source0 和 Sourcs1。
    * Source0：只包含了一个回调指针，不能主动触发事件。使用时需要先调用 `CFRunLoopSourceSignal` 将这个 Source 标记为待处理，然后手动调用 `CFRunLoopWakeUp` 来唤醒 RunLoop 处理该事件。
    * Source1：包含了一个 **mach_port** 和一个回调指针，被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 线程。
* **CFRunLoopTimerRef**：是基于事件的触发器，它和 `NSTimer` 是 `toll-free bridge` 的，可以混用。其中包括一个时间记录和一个回调指针。当加入到 RunLoop 时，RunLoop 会注册对应的时间点，当时间点到达的时候，RunLoop 会被唤醒来执行回调。
* **CFRunLoopObserverRef**：针对于 Observer 的一个引用，包含一个回调指针，状态发生变化时，Observer 能通过回调接受到变化。观测的状态通过枚举方式定义了一个状态机：

```Objective-C
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```

Source/Timer/Observer 统称为 ModeItem，一个 item 可以同时加入多个 mode ，但是 item 被重复加入同一个 Mode 是不会有效果的。这也符合数据结构中 `set` 的定义。如果 Mode 中的一个 Item 都没有，RunLoop 则会在该线程中直接退出，这也就是非主线程在构造后需要手动添加 Mode 并显示调用 Run 的原因。

## 23. 大概叙述一下 RunLoop 的整体流程？

继续推荐 ibireme 大佬的 [深入理解RunLoop
](https://blog.ibireme.com/2015/05/18/runloop/#base)。

![](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_1.png)

## 24. AutoreleasePool 是如何通过 RunLoop 工作的？

App 启动后，Core Foundation 会在主线程的 RunLoop 中注册两个 Observer，其回调都是 `_wrapRunLoopWithAutoreleasePoolHandler`。

第一个 Observer 监听的时间是 Entry，内部会调用 `_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()`，用来释放旧池创建新池。且优先级最高，保证重置操作在所有回调之前。

第二个 Observer 监听来能个事件，BeforeWaiting 时调用 `_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()` 用来重置释放池。且优先级最低，保证重置操作在所有回调之后。

主线程中也是如此，所以不用显式创建 `AutoreleasePool` 了。

## 25. `NSTimer`、`CADisplayLink` 是如何通过 RunLoop 工作的？

`NSTimer` 前文讲到就是 `CFRunLoopTimerRef`，他们是 TFB 桥接的。一个 `NSTimer` 注册到 RunLoop 后，会添加一个时间点记录。当事件到达且误差不超出 Tolerance （误差范围度），则执行 `CFRunLoopTimerRef` 中的回调方法。

`CADisplayLink` 是一个和屏幕刷新频率一致的定时器，其原理不尽相同。（暂时没有做过多的分析，但是知道内部操作了一个 Source。后续如果有机会代码分析可以补充上。）

## 26. 不手动指定 `autoreleasePool` 的前提下，一个 `autorelease` 对象在什么时候释放？例如在一个 `ViewController` 中的 `viewDidLoad` 中创建。

分两种情况：**手动干预释放时机**、**系统自动释放**。

1. 手动干预释放时机：指定 `autoreleasePool`，就是花括号作用域指定。
2. 系统自动释放。


由于 `autoreleasePool` 对象是加入 RunLoop 来进行重置的，从 App 启动到加载是完整的 RunLoop，然后会停下来等待用户交互，用户的每一次交互都会启动一次 RunLoop 来处理用户所有的点击事件和触摸事件。

当 `autoreleasePool` 销毁的同时，会将池内所有的对象发送 `release` 消息，此时会释放其中所有对象。如果在 `viewDidLoad` 中创建释放池，并添加对象，那么其将会在 `viewDidAppear` 方法执行前被销毁。

## 27. `autoreleasePool` 数据结构是怎样的？是如何管理和调度的？

推荐博客 [自动释放池的前世今生 - 深入解析 autoreleasepool](https://draveness.me/autoreleasepool)

`autoreleasePool` 是由多个 `AutoreleasePoolPage` 组成的一个链式线性表，这里我们来关注一下 `AutoreleasePoolPage` 的结构：

```C++
class AutoreleasePoolPage {
    magic_t const magic;        // 用于对当前 AutoreleasePoolPage 完整性校验
    id *next;                   // 指向下一节点
    pthread_t const thread;     // 保存了当前所在页的线程
    AutoreleasePoolPage * const parent; // 父亲节点
    AutoreleasePoolPage *child;         // 孩子节点
    uint32_t const depth;               
    uint32_t hiwat;
};
```

每一个 `AutoreleasePoolPage` 大小均为 `4096` 字节（`0x1000`）。

![](http://p632x9050.bkt.clouddn.com/ch03-1.png)

 每个 `AutoreleasePoolPage` 在内存栈中的图示：
 
 ![](http://p632x9050.bkt.clouddn.com/ch03-2.png)
 
 `POOL_SENTINEL` 是*哨兵对象*，是 `nil` 的宏。在每个 `AutoreleasePage` 被 `objc_autoreleasePoolPush` 的时候，会把一个 `POOL_SENTINEL` 添加到释放池的顶端。这个哨兵就好比是每个池的边界节点，当 `objc_autoreleasePoolPop` 调用时，就会像池中对象发送 `release` 消息，直到访问到第一个 `POOL_SENTINEL` 从而停止 `release` 操作。调度方面除了 `push`、`pop` 常规操作，还有 *满页情况(autoreleaseFullPage)* 和 *无页情况(autoreleaseNoPage)* 的处理，都很常规。
 
 ## 28. GCD 的队列(`dispatch_queue_t`) 分成哪两种？
 
 根据类型分成：
 
 1. 串行队列 - Serial Dispatch Queue：同一时间队列中只有一个任务在执行，每个任务只有在前一个任务执行完成后才能开始执行；
 2. 并行队列 - Concureent Dispatch Queue：这些任务会按照被添加的顺序开始执行。但是可以任意顺序。
 
根据作用特点划分：
1. 全局队列（属于并行队列）：不能与 barrier 栅栏搭配使用，barrier 只能与自定义的并行队列一起使用，否则 barrier 无法达到预期效果；如果和串行队列和 global 队列一起使用，barrier 表现与 `dispatch_sync` 一样。
2. 主队列（属于串行队列）：不能与 sync 同步方法一起使用，会造成死锁。

## 29. 如何用 GCD 同步若干个异步调用？

使用 Dispatch Group 来组合多个异步调用，然后通过 `dispatch_group_notify` 来获取所有异步调用的状态。例如用 URL 来加载多张图片，在下载完成后处理多张图片的代码示例：


```Objective-C
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
dispatch_group_async(group, queue, ^{ 
    // 加载图片一
});
dispatch_group_async(group, queue, ^{
    // 加载图片二
});
dispatch_group_async(group, queue, ^{
    // 加载图片三
});
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    // 处理图片
});
```

## 30. `dispatch_barrier_async` 的作用是什么？

在并行队列中，为了保证某些任务的顺序，需要等待一些任务完成后才能继续进行，使用 `barrier` 来等待之前任务完成，避免数据竞争等问题。`dispatch_barrier_async` 方法会的姑娘爱追加到 Concurrent Dispatch Queue 并行队列中的操作全部执行完成之后，在执行 `dispatch_barrier_async` 中的处理，在 `dispatch_barrier_async` 完成之后，并行队列恢复动作继续执行。

