
[TOC]

# iOS 面试基础题目

## 31. 如何手动触发一个 value 的 KVO 回调？

KVO 依赖于观察 `NSObject` 的两个方法：`willChangeValueForKey:` 和 `didChangeValueForKey`。这两个方法会在 `setter` 的赋值前后自动启用，没有必要显式写出。

```Objective-C
- (void)viewDidLoad {
   [super viewDidLoad];
   _now = [NSDate date];
   [self addObserver:self forKeyPath:@"now" options:NSKeyValueObservingOptionNew context:nil];
   NSLog(@"1");
   [self willChangeValueForKey:@"now"]; // “手动触发self.now的KVO”，必写。
   NSLog(@"2");
   [self didChangeValueForKey:@"now"]; // “手动触发self.now的KVO”，必写。
   NSLog(@"4");
}
```

## 32. KVO 实现原理？

先给出 KVO 官方的[解释文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOImplementation.html)。

关键词：**isa-swizzling**。当我对一个对象进行观察时，这时候 Runtime 会动态创建一个类，这个类继承自该对象自身的类，并重写了所要观察属性的 `setter` 方法。它会在赋值前后增加通知方法 `willChangeValueForKey` 和 `didChangevlueForKey`，然后原先对象的 isa 指针指向新创建的类。

这里有一篇 *mikeash* 大神的[经典文章](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)可以参考一下。

## 33.  `IBOutlet` 连出来的视图属性为什么可以被设置成 `weak`?

因为 `xib` 或者 `storyboard` 中的视图，可以当做对应的 View 已经有了一个强引用持有，所以只需要设置这个持有指针位 `weak` 即可。

另外，在 `storyboard` 中创建的 ViewController（xib 是不存在的），会同时生成一个 `_topLevelObjectsToKeeyAliveFromStoryboard` 的私有数组来强引用所有的 Top Level 对象，所以再拖出 `IBOutlet` 的声明可以声明成 weak。

## 34. Clang Extension 对于 Block 的扩展，其数据结构是怎样的？

一个 Block 在通过 `clang -rewrite-objc` 重写后，会变成以下数据结构：

```c
struct __block_impl {
	void *isa;
	int Flags;
	int Reserved;
	void *FuncPtr;
};

struct __outside_block_impl_0 {
    struct __block_impl impl;
    struct __outside_block_desc_0* Desc;
    __outside_block_impl_0(void *fp, struct __outside_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteGlobalBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static struct __outside_block_desc_0 {
	size_t reserved;
	size_t Block_size;
} __outside_block_desc_0_DATA = { 
	0, 
	sizeof(struct __outside_block_impl_0)
};

static void __outside_block_func_0(struct __outside_block_impl_0 *__cself) {
    /* block 的执行体 */
}
```

* **isa**：会指向一种 Block 类对象。在非 GC 的模式下只有三种：`_NSConcreteStackBlock`、`_NSConcreteGlobalBlock`、`_NSConcreteMallocBlock`。
* **Flags**：Block 的负载信息（引用计数和类型信息），按二进制位来存储。
* **Reserved**：保留字段。
* **FuncPtr**：指向 Block 执行体的指针。

## 35. Objective-C 中 Block 有几种类型？出现情况时什么样的？

* **_NSConcreteStackBlock**：栈类型，所处地址单元位于高地址。一般情况下声明的 Block 均为该类型，内部引入外部变量但不做赋值操作。
* **_NSConcreteMallocBlock**：堆类型。若需截获 `__block` 变量，改类型 Block 将出现。通过将对象和 Block 复制到堆上，保证了 `__block` 变量的作用域可扩展到 Block 内。这时会使用 `__forwarding` 指针来做关联，示意图如下。
* **_NSConcreteGloalBlock**：全局类型，所处地址单元位于低地址。两种出现时机：1. 在全局变量位置声明；2. Block 中不引入外部变量。

![](http://p632x9050.bkt.clouddn.com/ch04-1.png)

## 36. Associated Objects 存储方式

可以参考自己的博客 [浅谈Associated Objects](https://www.desgard.com/Associated-Objects/)。

![](http://7xwh85.com1.z0.glb.clouddn.com/img_2.png)

`AssociationsManager` 是一个顶级结构体，也是一个全局的单例，其中维护了 `spinlock_t` 锁和一个 `_map` 的 hashmap。这个 hashmap 的键为 `disguised_ptr_t` 的地址，通过它可以找到一个子 hashmap 里面存储了所有的 Associated Object。

## 37. 重载 Class 的 `+load` 方法时需不需要调父类？在自己 Class 的 `+load` 方法时能不能替换系统 framework （例如 UIKit）中的某个方法实现？

1. Runtime 负责按继承顺序递归调用，所有不用手动调 super。
2. 可以。因为动态链接过程中，所有依赖库的类是先与自己的类加载的。

## 38. 重载 `+load` 时需要手动添加 `@autoreleasepool` 么？想让一个类 `+load` 方法被调用是否需要在某个地方 `import` 这个文件？

1. 不需要，在 Runtime 调用 `+load` 方法前后，调用栈能够看到 `objc_autoreleasePoolPush()` 和 `objc_autoreleasePoolPop()` 两个方法的。说明系统自动调用。
2. 不需要，只要这个类的符号被编译到最后的可执行文件中，`+load` 方法就会被调用。

