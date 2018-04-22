# 《iOS Core Animation: Advanced Techniques》 读书笔记

## Questions:

- 隐式动画、显示动画的区别；

## Tips
- `UIView`可以处理触摸事件，可以支持基于Core Graphics绘图，可以做仿射变换（例如旋转或者缩放），或者简单的类似于滑动或者渐变的动画。
- `-drawRect:`默认没有实现，重写方法会为视图分配寄宿图，浪费CPU。


## CALayer
### Property
- `contents`  
显示`CGImage`.
- `contentGravity`  
决定内容在图层的边界中怎么对齐.
- `contentsScale`  
每个点的像素值.
- `contentsRect`  
显示子域.
- `doubleSided`  
默认为`YES`，如果设置为`NO`，那么当图层正面从相机视角消失的时候，它将不会被绘制.

### 重绘流程
有一个可选的`delegate`属性，实现了`CALayerDelegate`协议.
1. 请求代理给一个寄宿图来显示:
`- (void)displayLayer:(CALayerCALayer *)layer;`
2. 尝试调用:
`- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx;`

### 坐标系
不同图层中的坐标转换:

```
- (CGPoint)convertPoint:(CGPoint)point fromLayer:(CALayer *)layer;
- (CGPoint)convertPoint:(CGPoint)point toLayer:(CALayer *)layer;
- (CGRect)convertRect:(CGRect)rect fromLayer:(CALayer *)layer;
- (CGRect)convertRect:(CGRect)rect toLayer:(CALayer *)layer;
```

### Response Chain
```
- (BOOL)containsPoint:(CGPoint)p;   // 坐标需要转换成各坐标系下
- (CALayer *)hitTest:(CGPoint)p;    // 测算的顺序严格依赖于图层树当中的图层顺序,zPosition属性可以明显改变屏幕上图层的顺序,但不能改变事件传递的顺序
```

### 布局
`CALayerDelegate`代理下的：

```
- (void)layoutSublayersOfLayer:(CALayer *)layer;
```
调用时机：当图层的`bounds`发生改变，或者图层的`-setNeedsLayout`方法被调用。不能做到自适应屏幕旋转。

### 阴影
- `shadowOpacity`  
当图层拥有多个子图层，每个图层还拥有透明寄宿图，计算阴影很消耗资源。
- `shadowPath`  
`shadowPath`是`CGPathRef`类型，可以通过`UIBezierPath`绘制曲线赋值最终绘制阴影。

### 拉伸过滤——过滤滤波
`minification`（缩小图片）和`magnification`（放大图片）

```
kCAFilterLinear     // 双线性滤波算法（默认）
kCAFilterNearest    // 最近过滤
kCAFilterTrilinear  // 三线性滤波算法
```

### 透明度
`shouldRasterize`
图层及其子图层都会被整合成一个整体的图片，需要用`rasterizationScale`去匹配屏幕，防止Retina屏幕像素化。


## 布局
![旋转一个视图或者图层之后的frame属性](media/15226548716427/15228241732455.jpg)


## 变换
### AffineTransform仿射变换
“仿射”的意思是无论变换矩阵用什么值，图层中平行的两条线在变换之后任然保持平行。
`UIView`的`transform`属性是一个`CGAffineTransform`类型，是一个可以和二维空间向量（例如`CGPoint`）做乘法的3X2的矩阵：
![](media/15226548716427/15238458786760.jpg)
#### API：

```
CGAffineTransformMakeRotation(CGFloat angle)
CGAffineTransformMakeScale(CGFloat sx, CGFloat sy)
CGAffineTransformMakeTranslation(CGFloat tx, CGFloat ty)
```

#### 混合变换

```
CGAffineTransformIdentity   // 空值
CGAffineTransformRotate(CGAffineTransform t, CGFloat angle)
CGAffineTransformScale(CGAffineTransform t, CGFloat sx, CGFloat sy)
CGAffineTransformTranslate(CGAffineTransform t, CGFloat tx, CGFloat ty)

// 合并两个变换
CGAffineTransformConcat(CGAffineTransform t1, CGAffineTransform t2);
```

### 3D变换
`CALayer`的`transform`属性是`CATransform3D`，是一个可以在3维空间内做变换的4x4的矩阵：
![](media/15226548716427/15238621946214.jpg)
#### API：

```
CATransform3DMakeRotation(CGFloat angle, CGFloat x, CGFloat y, CGFloat z)
CATransform3DMakeScale(CGFloat sx, CGFloat sy, CGFloat sz) 
CATransform3DMakeTranslation(Gloat tx, CGFloat ty, CGFloat tz)
```

分别绕不同的轴的旋转方向：
![](media/15226548716427/15238627245951.jpg)

#### 透视投影
在真实世界中，当物体远离我们的时候，由于视角的原因看起来会变小，理论上说远离我们的视图的边要比靠近视角的边跟短，但实际上并没有发生，而我们当前的视角是等距离的，也就是在3D变换中任然保持平行，和之前提到的仿射变换类似。
在等距投影中，远处的物体和近处的物体保持同样的缩放比例，这种投影也有它自己的用处（例如建筑绘图，颠倒，和伪3D视频），但当前我们并不需要。

为了做一些修正，我们需要引入投影变换（又称作z变换）来对除了旋转之外的变换矩阵做一些修改，Core Animation并没有给我们提供设置透视变换的函数，因此我们需要手动修改矩阵值，幸运的是，很简单：
`CATransform3D`的透视效果通过一个矩阵中一个很简单的元素来控制：`m34`。`m34`用于按比例缩放X和Y的值来计算到底要离视角多远。

`m34`的默认值是0，我们可以通过设置`m34`为-1.0 / `d`来应用透视效果，`d`代表了想象中视角相机和屏幕之间的距离，以像素为单位，那应该如何计算这个距离呢？实际上并不需要，大概估算一个就好了。
因为视角相机实际上并不存在，所以可以根据屏幕上的显示效果自由决定它的防止的位置。通常500-1000就已经很好了，但对于特定的图层有时候更小后者更大的值会看起来更舒服，减少距离的值会增强透视效果，所以一个非常微小的值会让它看起来更加失真，然而一个非常大的值会让它基本失去透视效果。

#### 灭点
灭点应该聚在屏幕中点，或者至少是包含所有3D对象的视图中点。

Core Animation定义了这个点位于变换图层的`anchorPoint`（通常位于图层中心，但也有例外，见第三章）。这就是说，当图层发生变换时，这个点永远位于图层变换之前`anchorPoint`的位置。
当改变一个图层的`position`，你也改变了它的灭点，做3D变换的时候要时刻记住这一点，当你视图通过调整`m34`来让它更加有3D效果，应该首先把它放置于屏幕中央，然后通过平移来把它移动到指定位置（而不是直接改变它的`position`），这样所有的3D图层都共享一个灭点。

#### `sublayerTransform`属性
可以一次性对包含这些图层的容器做变换，于是所有的子图层都自动继承了这个变换方法。同时它会带来另一个显著的优势：灭点被设置在容器图层的中点，从而不需要再对子图层分别设置了。这意味着你可以随意使用position和frame来放置子图层，而不需要把它们放置在屏幕中点，然后为了保证统一的灭点用变换来做平移。


## 专用图层
### `CAShapeLayer`
通过矢量图形而不是bitmap来绘制。优点如下：

- 渲染快速。CAShapeLayer使用了硬件加速，绘制同一图形会比用Core Graphics快很多。
- 高效使用内存。一个CAShapeLayer不需要像普通CALayer一样创建一个寄宿图形，所以无论有多大，都不会占用太多的内存。
- 不会被图层边界剪裁掉。一个CAShapeLayer可以在边界之外绘制。你的图层路径不会像在使用Core Graphics的普通CALayer一样被剪裁掉（如我们在第二章所见）。
- 不会出现像素化。当你给CAShapeLayer做3D变换时，它不像一个有寄宿图的普通图层一样变得像素化。

### `CATextLayer`
`CATextLayer`也要比`UILabel`渲染得快得多。使用了Core text，并且渲染得非常快。

> Tips:
> 重写`UIView`的`+layerClass`方法使得在创建的时候能返回一个不同的图层子类

### `CATransformLayer`
它不能显示它自己的内容。只有当存在了一个能作用与子图层的变换它才真正存在。`CATransformLayer`并不平面化它的子图层，所以它能够用于构造一个层级的3D结构。

### `CAGradientLayer`
用来生成两种或更多颜色平滑渐变的。用Core Graphics复制一个CAGradientLayer并将内容绘制到一个普通图层的寄宿图也是有可能的，但是CAGradientLayer的真正好处在于绘制使用了硬件加速。

### `CAReplicatorLayer`
为了高效生成许多相似的图层。它会绘制一个或多个图层的子图层，并在每个复制体上应用不同的变换。

### `CAScrollLayer`
`-scrollToPoint:`方法，它自动适应`bounds`的原点以便图层内容出现在滑动的地方。注意，这就是它做的所有事情。前面提到过，Core Animation并不处理用户输入，所以`CAScrollLayer`并不负责将触摸事件转换为滑动事件，既不渲染滚动条，也不实现任何iOS指定行为例如滑动反弹（当视图滑动超多了它的边界的将会反弹回正确的地方）。

### `CATiledLayer`
将大图分解成小片然后将他们单独按需载入。
256*256是`CATiledLayer`的默认小图大小。

```
-drawLayer:inContext:       // 不是线程安全的
```

### `CAEmitterLayer`
一个高性能的粒子引擎，被用来创建实时例子动画如：烟雾，火，雨等等这些效果。

`CAEMitterCell`的属性基本上可以分为三种：

- 这种粒子的某一属性的初始值。比如，color属性指定了一个可以混合图片内容颜色的混合色。在示例中，我们将它设置为桔色。
- 粒子某一属性的变化范围。比如emissionRange属性的值是2π，这意味着例子可以从360度任意位置反射出来。如果指定一个小一些的值，就可以创造出一个圆锥形
- 指定值在时间线上的变化。比如，在示例中，我们将alphaSpeed设置为-0.4，就是说例子的透明度每过一秒就是减少0.4，这样就有发射出去之后逐渐小时的效果。

### `CAEAGLLayer`
用来显示任意的OpenGL图形。

## 隐式动画
动画执行的时间取决于当前事务的设置，动画类型取决于图层行为。
事务实际上是Core Animation用来包含一系列属性动画集合的机制，任何用指定事务去改变可以做动画的图层属性都不会立刻发生变化，而是当事务一旦提交的时候开始用一个动画过渡到新值。
Core Animation在每个run loop周期中自动开始一次新的事务，即使你不显式的用`[CATransaction begin]`开始一次事务，任何在一次run loop循环中属性的改变都会被集中起来，然后做一次0.25秒的动画。

当`CALayer`的属性被修改时候，它会调用`-actionForKey:`方法，传递属性的名称。剩下的操作都在`CALayer`的头文件中有详细的说明，实质上是如下几步：

- 图层首先检测它是否有委托，并且是否实现`CALayerDelegate`协议指定的`-actionForLayer:forKey`方法。如果有，直接调用并返回结果。
- 如果没有委托，或者委托没有实现`-actionForLayer:forKey`方法，图层接着检查包含属性名称对应行为映射的`actions`字典。
- 如果`actions`字典没有包含对应的属性，那么图层接着在它的`style`字典接着搜索属性名。
- 最后，如果在`style`里面也找不到对应的行为，那么图层将会直接调用定义了每个属性的标准行为的`-defaultActionForKey:`方法。
通过返回`nil`禁用隐式动画。

```
[CATransaction setDisableActions:YES];      // 另外一种方式禁用隐式动画
```

### 呈现与模型
每个图层属性的显示值都被存储在一个叫做呈现图层的独立图层当中，他可以通过`-presentationLayer`方法来访问。这个呈现图层实际上是模型图层的复制，但是它的属性值代表了在任何指定时刻当前外观效果。换句话说，你可以通过呈现图层的值来获取当前屏幕上真正显示出来的值。
呈现树通过图层树中所有图层的呈现图层所形成。注意呈现图层仅仅当图层首次被提交（就是首次第一次在屏幕上显示）的时候创建，所以在那之前调用`-presentationLayer`将会返回`nil`。
呈现图层work on:

- 如果你在实现一个基于定时器的动画，而不仅仅是基于事务的动画，这个时候准确地知道在某一时刻图层显示在什么位置就会对正确摆放图层很有用了。
- 如果你想让你做动画的图层响应用户输入，你可以使用`-hitTest:`方法来判断指定图层是否被触摸，这时候对呈现图层而不是模型图层调用`-hitTest:`会显得更有意义，因为呈现图层代表了用户当前看到的图层位置，而不是当前动画结束之后的位置。

## 显示动画
`CAAnimation`提供了一个计时函数，一个委托（用于反馈动画状态）以及一个`removedOnCompletion`，用于标识动画是否该在结束后自动释放（默认YES，为了防止内存泄露）。`CAAnimation`同时实现了一些协议，包括`CAAction`（允许`CAAnimation`的子类可以提供图层行为），以及`CAMediaTiming`。
### 属性动画`CAPropertyAnimation`
属性动画通过指定动画的`keyPath`作用于图层的某个单一属性，并指定了它的一个目标值，或者一连串将要做动画的值。属性动画只对图层的可动画属性起作用，所以如果要改变一个不能动画的属性（比如图片），或者从层级关系中添加或者移除图层，属性动画将不起作用。

#### 基础动画`CABasicAnimation`

```
id fromValue    // id可以指定很多不同的属性类型，包括数字类型，矢量，变换矩阵，甚至是颜色或者图片。
id toValue 
id byValue
```

用于CAPropertyAnimation的一些类型转换：

| Type | Object Type | Code Example |
| --- | --- | --- |
| CGFloat | NSNumber | id obj = @(float);|
| CGPoint | NSValue	| id obj = [NSValue valueWithCGPoint:point);|
| CGSize | NSValue | id obj = [NSValue valueWithCGSize:size);|
| CGRect | NSValue	id | obj = [NSValue valueWithCGRect:rect);|
| CATransform3D | NSValue | id obj = [NSValue valueWithCATransform3D:transform);|
| CGImageRef | id | id obj = (\_\_bridge id)imageRef;|
| CGColorRef | id | id obj = (__bridge id)colorRef;|

#### `CAAnimationDelegate`
可以通过`animationKeys`或者KVC协议的`-setValue:forKey:`和`-valueForKey:`区分一个图层中的不同动画。
真机上，回调方法在动画完成之前已经被调用了，但不能保证这发生在属性动画返回初始状态之前。

#### 关键帧动画`CAKeyframeAnimation`
作用于单一的一个属性，但是和`CABasicAnimation`不一样的是，它不限制于设置一个起始和结束的值，而是可以根据一连串随意的值来做动画。
`CAKeyframeAnimation`并不能自动把当前值作为第一帧（就像`CABasicAnimation`那样把`fromValue`设为`nil`）。动画会在开始的时候突然跳转到第一帧的值，然后在动画结束的时候突然恢复到原始的值。所以为了动画的平滑特性，我们需要开始和结束的关键帧来匹配当前属性的值。

```
CAKeyFrameAnimation.rotationMode = kCAAnimationRotateAuto;  // 图层将会根据曲线的切线自动旋转
```

#### 虚拟属性
用`transform.rotation`而不是`transform`做动画的好处如下：

- 我们可以不通过关键帧一步旋转多于180度的动画。
- 可以用相对值而不是绝对值旋转（设置`byValue`而不是`toValue`）。
- 可以不用创建`CATransform3D`，而是使用一个简单的数值来指定角度。
- 不会和`transform.position`或者`transform.scale`冲突（同样是使用关键路径来做独立的动画属性）。

### 过渡
- `CALayer`

```
CATransition.type
kCATransitionFade 
kCATransitionMoveIn 
kCATransitionPush 
kCATransitionReveal

CATransition.subtype
kCATransitionFromRight 
kCATransitionFromLeft 
kCATransitionFromTop 
kCATransitionFromBottom
```

- `UIView`

```
+transitionFromView:toView:duration:options:completion:
+transitionWithView:duration:options:animations:

options:
UIViewAnimationOptionTransitionFlipFromLeft 
UIViewAnimationOptionTransitionFlipFromRight
UIViewAnimationOptionTransitionCurlUp 
UIViewAnimationOptionTransitionCurlDown
UIViewAnimationOptionTransitionCrossDissolve 
UIViewAnimationOptionTransitionFlipFromTop 
UIViewAnimationOptionTransitionFlipFromBottom
```

#### 自定义过渡
过渡动画做基础的原则就是对原始的图层外观截图，然后添加一段动画，平滑过渡到图层改变之后那个截图的效果。
`CALayer`有一个`-renderInContext:`方法，可以通过把它绘制到Core Graphics的上下文中捕获当前内容的图片，然后在另外的视图中显示出来。如果我们把这个截屏视图置于原始视图之上，就可以遮住真实视图的所有变化，于是重新创建了一个简单的过渡效果。
用`CABasicAnimation`需要对图层的变换和不透明属性创建单独的动画，然后当动画结束的时候在`CAAnimationDelegate`中把`coverView`从屏幕中移除。

## 图层时间`CAMediaTiming`(protocol)

### `fillMode`

```
kCAFillModeForwards
kCAFillModeBackwards
kCAFillModeBoth
kCAFillModeRemoved
```

> Tips:
> 对`CALayer`或者`CAAnimationGroup`调整`duration`和`repeatCount`/`repeatDuration`属性并不会影响到子动画。但是`beginTime`，`timeOffset`和`speed`属性将会影响到子动画。
> 给图层添加一个`CAAnimation`实际上是给动画对象做了一个不可改变的拷贝，所以对原始动画对象属性的改变对真实的动画并没有作用。相反，直接用`-animationForKey:`来检索图层正在进行的动画可以返回正确的动画对象，但是修改它的属性将会抛出异常。

### 手动动画
通过设置`speed`为0，可以禁用动画的自动播放，然后来使用`timeOffset`来来回显示动画序列。

## 缓冲

### 动画速度
恒定速度的动画我们称之为“线性步调"。

### `CAMediaTimingFunction`
`UIView`默认`UIViewAnimationOptionCurveEaseInOut`，新建的`CAAnimation`默认`kCAMediaTimingFunctionLinear`。
`CAMediaTimingFunction`使用了一个叫做三次贝塞尔曲线的函数，它只可以产出指定缓冲函数的子集。
`-getControlPointAtIndex:values:`可以用来检索曲线的控制点。曲线的起始和终点始终是{0, 0}和{1, 1}，只需要检索曲线的第二个和第三个点。

实现一个橡胶球掉落到坚硬的地面的场景，当开始下落的时候，它会持续加速知道落到地面，然后经过几次反弹，最后停下来。时间-改变的量函数图如图：
![10.5](media/15226548716427/10.5.jpeg)
可以用如下几种方法：

- 用`CAKeyframeAnimation`创建一个动画，然后分割成几个步骤，每个小步骤使用自己的计时函数。
- 使用定时器逐帧更新实现动画。

### 自动化带缓冲的关键帧动画



