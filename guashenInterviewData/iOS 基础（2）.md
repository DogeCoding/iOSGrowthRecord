[TOC]

# iOS 面试基础题目

## 11. `@synthesize` 合成实例变量的规则是什么？假如 `@property` 存在一个名为 `foo` 属性，我在声明一个 `@property` 名为 `_foo`，会导致什么结果？

有两种情况：

1. 如果我显式写明了 `@synthesize` 规则，且 `@synthesize foo = ***`，其中 `**` 不是 `_foo` 那么可以同时存在。
2. 如果未显式写明 `@synthesize` 规则，属性则自动与 `_ + 属性名` 关联成员变量，此时 xcode 会报出 warning ，且不会自动合成新变量。

![](http://p632x9050.bkt.clouddn.com/ch02-1.jpeg)

## 12. 什么情况下不会 autosynthesize ?

1. 重写了 `setter` 和 `getter` 时
2. 重写了只读属性的 `getter` 
3. 使用了 `@dynamic`
4. 在 `@protocol` 中定义了属性
5. 在 `category` 中定义了属性
6. overrite 了属性

这里尤其注意到，当子类重载了父类的属性时，我们需要用 `@synthesize` 来手动合成 `ivar`。



[When should I use @synthesize explicitly?](https://stackoverflow.com/questions/19784454/when-should-i-use-synthesize-explicitly/19821816#19821816)

## 13. ObjC 中向一个对象发送消息 `[obj foo]` 和 `objc_msgSend()` 方法有什么关系？

查阅自己写的博客：

所谓发送消息 `[obj foo]` 在编译后其实就是 `objc_msgSend()` 函数调用。

```Objective-C
id retureValue = [DGObject foo];

// 等效于
id retureValue = objc_msgSend(DGObject, @selector(test));
```

`objc_msgSend` 方法原型：

```C
void objc_msgSend(id self, SEL cmd, ...)
```

传入 `self` 指针和 `SEL` 选择子即可。

## 14. `unrecognized selector` 异常出现在什么情况下？消息转发的三个阶段？

调用改对象的某个方法却缺少具体的实现的时候。

关于消息转发，自己有两篇博客可以参考：[objc_msgSend消息传递学习笔记 - 对象方法消息传递流程](http://www.desgard.com/objc_msgSend1/)，[objc_msgSend消息传递学习笔记 - 消息转发](http://www.desgard.com/objc_msgSend2/)。

**1. 动态方法解析过程**

当 `objc_msgSend` 无法在 *Fast Map* 中根据选择子查询到对应名称的方法时，进入 `resolveInstanceMethor` 过程。在 `resolveInstanceMethod` 过程中，进入入口方法 `_class_resolveMethod`，先根据 isa 的指向来判断是类方法还是成员方法，从而调用 `resolveInstanceMethod` 或者 `resolveClassMethod`。主体流程中，执行 `lookUpImpOrNil` 方法在方法列表中来搜索 IMP 指针。若仍未命中，继续进入 **备援接受阶段**。

**2. Forwarding 备援接收**

如果选择子未命中，则继续将其传递给其他接受者来处理。这一块代码调用的主要方法为 `forwardingTargetForSelector: (SEL)selector`。在 Core Foundation 中这一方法的实现被隐藏，暴露出来的 `__forwarding__` 的核心方法 `objc_msgSend_stret` 也被隐藏。

**3. Message Dispatch System 消息派发**

在 Forwarding 步骤中最后会创建一个 `NSInvocation` 对象，其中包括了对应的选择子、target 主要参数。倘若自下而上始终没有发现可处理改消息的方式，则调用 `doesNotRecognizeSelector` 从而抛出异常。

![](http://7xwh85.com1.z0.glb.clouddn.com/Desktop.png)

## 15. ObjC 对象的 Memory Layout？与 Swift 的区别？

可参考自己写的博客：[用 isa 承载对象的类信息](http://www.desgard.com/isa/)

[objc4-680.tar.gz]() 版本的 Runtime 中其对象和类的数据结构如下描述：

```c
struct objc_object {
private:
    isa_t isa;
}

struct objc_class : objc_object {
    // Class ISA;
    Class superclass; 	// 父类引用
    cache_t cache;		// 用来缓存指针和虚函数表
    class_data_bits_t bits; // class_rw_t 指针加上 rr/alloc 标志
}
```

`objc_class` 通过继承关系保留 `isa` 指针。联合体 `union isa_t` 的数据结构如下：

```c
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
    uintptr_t bits;
    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; 
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
    };
};
```


区域名 | 代表信息
---|---
indexed| 0 表示普通的 isa 指针，1 表示使用优化，存储引用计数
has_assoc | 表示该对象是否包含 associated object，如果没有，则析构时会更快
has_cxx_dtor |	表示该对象是否有 C++ 或 ARC 的析构函数，如果没有，则析构时更快
shiftcls|	类的指针
magic	|固定值，用于在调试时分辨对象是否未完成初始化
weakly_referenced|	表示该对象是否有过 weak 对象，如果没有，则析构时更快
deallocating|	表示该对象是否正在析构
has_sidetable_rc|	表示该对象的引用计数值是否过大无法存储在 isa 指针
extra_rc|	存储引用计数值减一后的结果

给出 `class` 和 `metaclass` 的关系图：

![](http://7xwh85.com1.z0.glb.clouddn.com/isa_metaclass.png)

## 16. ObjC 对象的 `isa` 指针指向什么？有什么作用？

* 普通成员对象的 isa 指针指向其类对象
* 类对象 isa 指针指向其 metaClass
* metaClass 的 isa 指针有两种情况：1. `NSObject` 类对象的 isa 指向自己形成自环；2. 非 `NSObject` 类对象的 isa 指向其 `superclass` 类对象。

`isa` 指针作用，用于消息转发自上而下的查询流程，用于 Memory Layout 中的访问初始化操作，将所有的 Class 由于 `NSObject` 的存在而组织成图形链式结构，更加灵活。

## 17. 关于 `self` 和 `super` 关键字的理解？

下面的代码输出结果是什么？

```Objective-C
@implementation Son : Father
- (id)init
{
   self = [super init];
   if (self) {
       NSLog(@"%@", NSStringFromClass([self class]));
       NSLog(@"%@", NSStringFromClass([super class]));
   }
   return self;
}
@end
```

最终会输出：

```shell
NSStringFromClass([self class]) = Son
NSStringFromClass([super class]) = Son
```

`self` 是指向当前类实例的一个指针，但是 `super` 却是一个 *Magic Keyword*，他是一个编译器的标识符，与 `self` 指向同一消息接受者。`super` 关键字会告诉编译器当执行 `[super class]` 代码的时候，从父类发方法中执行，但是消息的对象都是 `Son *obj` 这个实例，在转换方法的时候同样的都会变成：

```Objective-C
objc_msgSend(obj, @selector(class));
```

## 18. `_objc_msgForward` 函数是做什么的，直接调用它将会发生什么？

`_objc_msgForward` 小心转发过程会设计到以下方法：

1. `resolveInstanceMethod:` 方法 (或 `resolveClassMethod:`)。

允许用户在此时为该 Class 动态添加实现。如果有实现了，则调用并返回YES，那么重新开始 `objc_msgSend` 流程。这一次对象会响应这个选择器，一般是因为它已经调用过 `class_addMethod`。如果仍没实现，继续下面的动作。

2. `forwardingTargetForSelector:` 方法

尝试找到一个能响应该消息的对象。如果获取到，则直接把消息转发给它，返回非 `nil` 对象。否则返回 `nil` ，继续下面的动作。注意，这里不要返回 `self` ，否则会形成死循环。

3. `methodSignatureForSelector:` 方法

调用 `methodSignatureForSelector:` 方法，尝试获得一个方法签名。如果获取不到，则直接调用 `doesNotRecognizeSelector` 抛出异常。如果能获取，则返回非 `nil`：创建一个 `NSlnvocation` 并传给 `forwardInvocation:`。

4. `forwardInvocation:` 方法

将第3步获取到的方法签名包装成 `Invocation` 传入，如何处理就在这里面了，并返回非 `nil`。

5. `doesNotRecognizeSelector:` 方法

默认的实现是抛出异常。如果第3步没能获得一个方法签名，执行该步骤。

## 19. RunLoop 和线程的关系？

RunLoop 是依托于线程存在的，每个线程，包括主线程都有对应的 RunLoop。除了主线程以外，其他线程的 RunLoop 都需要手动调用 Run 方法来开启，当线程结束后，RunLoop 由线程自行释放。

## 20. RunLoop 的 Mode 作用。

Mode 用来控制事件在运行循环中的优先级，分成以下几种：

* `NSRunLoopCommonModes`：一组 RunLoop Mode 集合，它包括了 `NSDefaultRunLoopMode`、`NSTaskDeathCheckMode`、`UITrackingRunLoopMode`，可以使用 `CFRunLoopAddCommonMode` 向其中添加自定义 Mode。
    * `NSDefaultRunLoopMode`（`kCFRunLoopDefaultMode`）：默认 Mode，空闲状态；
    * `NSTaskDeathCheckMode`
    * `UITrackingRunLoopMode`
* `UITrackingRunLoopMode`（`ScrollView`）滑动交互状态；
* `UIInitializationRunLoopMode`：启动状态；
* `NSConnectionReplyMode`：监听 `NSConnection` 对象状态；
* `NSModalPanelRunLoopMode`：在 Model Panel 情况下区分事件（macOS 开发）；
* `GSEventReceiveRunLoopMode`：接受系统时间，私有类；

开放的 RunLoop 只有以下两种：

* NSDefaultRunLoopMode（kCFRunLoopDefaultMode）
* NSRunLoopCommonModes（kCFRunLoopCommonModes）