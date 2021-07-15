---
layout: post
title: iOS中的AutoResizing和AutoLayout
image: 5.jpg
date: 2020-12-16 20:02:00 +0200
tags: [iOS, Objective-C]
categories: iOS
---

An introduction and explanation of the UI layout approach: AutoResizing and AutoLayout, in iOS developing.

***
# AutoResizing、AutoLayout

在进行UI开发的时候，布局经常是一个非常重要的问题。

现在的手机包括iPhone在内，都有不同的屏幕尺寸，同时还存在屏幕可以旋转的情况，所以在实现UI时我们一般不能去像AbsoluteLayout那样写死组件的位置，而是需要更灵活、适应性强的布局工具，来让我们的界面可以和不同大小和比例的屏幕适配。获取屏幕的尺寸再通过计算来设置绝对布局的数值也是一种方法，但是iOS为我们提供了更好的自动布局方式来实现对组件相对位置的控制。

在iOS开发中，AutoResizing和AutoLayout这两种机制都可以被用来配置视图的自动布局。在《Programming iOS 6》这本书里是这么定义这两个概念的：

**Autoresizing**

>Autoresizing is a matter of conceptually assigning a subview “springs and struts.” A spring can stretch; a strut can’t. Springs and struts can be assigned internally or externally. Thus you can specify (using internal springs and struts) whether and how the view can be resized, and (using external springs and struts) whether and how the view can be repositioned.
>
>Autoresizing是在概念上指定一个子视图的”弹簧和支架“。弹簧可以拉伸，而支架是固定的。可以在内部或者外部为子视图指定弹簧和支柱。因此，你可以用它们来确定（在父视图变化时）**是否、如何调整视图的大小，以及是否、如何重新定位视图。**

**AutoLayout**

> Autolayout, depends on the constraints of views. A constraint (an instance of NSLayoutConstraint) is much more sophisticated than the "autoresizingMask" it’s a full-fledged object with numeric values, and can describe a relationship between any two views (not just a subview and its superview).
>
> Autolayout取决于视图的”约束(constraints)“。约束是`NSLayoutConstraint`的一个实例，比autoresizingMask复杂得多，他是一个完整的带有数值的对象，可以描述**任意两个子视图之间的关系（不仅仅是子视图和父视图之间）。**

概括来说就是：

- AutoResizing可以决定子视图是否、如何跟随**父视图**的变化而进行变化。
- AutoLayout可以控制任意**两个子视图之间**的位置关系。

## Autoresizing

Autoresizing是iOS较早期对页面布局适配屏幕的一种解决方案。顾名思义，就是“自动调整大小”的意思。也就是说当父视图的大小形状改变时，子视图也自动的进行调整。

<img src="image/Autoresizing.png" alt="Autoresizing" style="zoom:70%;" />

从StoryBoard中配置Autoresizing的这部分可以比较清晰的看出它是怎样控制一个视图的大小位置的。

我们可以控制左侧正方形中显示的六个维度是成为“弹簧”还是成为“支架”。外侧的四条线代表子视图边缘和父视图边缘的距离关系，中间的两条线代表子视图的高度和宽度。我们可以简单的理解成选中的线会成为“支架”，而未选中的会变成“弹簧”。当我们把鼠标放到右侧这张小图片上时，它会动态的显示出按照目前的设置，子视图在父视图大小变化时会发生什么，非常直观。

如果想用代码来配置Autoresizing，我们需要配置一个UIView的属性，叫做`AutoresizingMask`，它的类型是一个叫`UIViewAutoresizing`的枚举类。它的枚举值有如下几种

- `UIViewAutoresizingNone`：不进行自动布局（默认）
- `UIViewAutoresizingFlexibleLeftMargin`：左侧与父视图距离弹性调整，右侧距离保持不变
- `UIViewAutoresizingFlexibleRightMargin`：右侧与父视图距离弹性调整，左侧保持不变
- `UIViewAutoresizingFlexibleTopMargin`：上方与父视图距离弹性调整，下方保持不变
- `UIViewAutoresizingFlexibleBottomMargin`：下方与父视图距离弹性调整，上方保持不变
- `UIViewAutoresizingFlexibleWidth`：弹性调整自身宽度，与父视图左右距离保持不变
- `UIViewAutoresizingFlexibleHeight`：弹性调整自身高度，与父视图上下距离保持不变

这几种类型可以组合使用，产生的效果和在StoryBoard中配置的效果是一模一样的。不过要注意不要配出相互冲突的组合。

在视图较为简单、子视图较少、约束关系可以仅仅局限于父子视图之间的时候，我们就可以用Autoresizing来轻松的配置子视图的动态大小和位置。

但是，Autoresizing的局限性也非常明显：它只能在**父子视图**之间产生约束关系，而不能在**子视图之间**进行约束。因此，当我们的子视图非常多，而对相对关系的要求比较复杂的时候，仅仅用它可能已经无法满足我们的需求了。这时，我们就需要使用更强大的工具：AutoLayout了。

## AutoLayout

### 概念

AutoLayout直译为自动布局。顾名思义，它不仅仅是对组件的大小进行调整，而是可以使整个布局都自动进行我们想要的调整。它就是为了代替功能不够强大的Autoresizing而出现的，并且是现在广泛使用的布局工具。正如上面的定义中提到的，AutoLayout可以**描述任意两个子视图之间**的关系，并且他也是对组件的**相对位置**进行设置。官方是这样描述的：AutoLayout是一个基于约束的，描述性的布局系统。

在配置AutoLayout时，其实我们就是在配置一个一个的**“约束”**，一般来说，一个约束可以描述一个组件的某一侧的位置或某个维度的大小，这个描述通常是相对其他组件而言的，例如<u>“组件A的左侧和组件B相隔30.0pt”</u>，<u>“组件A的宽度总是等于组件C的宽度”</u>，或是<u>“组件B的顶部总是与组件C的底部相隔20.0pt”</u>，等等。

在[Apple Developer](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/LayoutUsingStackViews.html#//apple_ref/doc/uid/TP40010853-CH11-SW1)官网上给出了非常详细的指导和例子：如果想设计一个大致如下图的布局，我们要设定的约束如下：

> <img src="https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/Art/dynamic_stack_view_2x.png" alt="AutoLayout" style="zoom:50%;" />
>
> 1. `Scroll View.Leading = Superview.LeadingMargin`
> 2. `Scroll View.Trailing = Superview.TrailingMargin`
> 3. `Scroll View.Top = Superview.TopMargin`
> 4. `Bottom Layout Guide.Top = Scroll View.Bottom + 20.0`
> 5. `Stack View.Leading = Scroll View.Leading`
> 6. `Stack View.Trailing = Scroll View.Trailing`
> 7. `Stack View.Top = Scroll View.Top`
> 8. `Stack View.Bottom = Scroll View.Bottom`
> 9. `Stack View.Width = Scroll View.Width`

注意上面这些不是代码，只是对布局约束的设计。

### 设置

有了计划之后我们就可以来配置这些约束了。既可以选择在StoryBoard中直接设置，也可以用代码来进行设置。

在StoryBoard中，我们可以直接通过右下角的这些工具栏为各个组件添加约束：

<img src="image/AutoLayout2.png" alt="AutoLayout2" style="zoom:50%;" /><img src="image/AutoLayout.png" alt="AutoLayout" style="zoom:41%;" /> 

除了用StoryBoard之外，用代码来进行配置可能会更加直观。

不过要注意的是，既然AutoLayout的功能是替代Autoresizing的，那么为了不造成冲突，我们在使用的时候就要先取消Autoresizing的功能。在用代码进行配置时，我们可以为每个组件添加这样一行代码：

```swift
// Swift
titleLabel.translatesAutoresizingMaskIntoConstraints = false
```

这个方法`translatesAutoresizingMaskIntoConstraints`意为将`autoresizingMask`转化为约束。既然我们自己要手动添加约束来进行自动布局，那么我们就不能再使用autoresizingMask配置的布局了。把这个值设置为`false`（OC中是`NO`）就可以忽略掉这些我们不想要的约束，而仅仅使用我们配置的AutoLayout约束。

我们可以使用类似如下代码来配置约束：

```Swift
// 调用这个方法传入一个Constraint数组
NSLayoutConstraint.activate([
    //    在这里放入各个view的Constraint
  //    1. 设置宽高
  //        - 设置为常数：
    titleLabel.widthAnchor.constraint(equalToConstant: 128.0),
    titleLabel.heightAnchor.constraint(equalToConstant: 32.0),
  //        - 等于别的view的宽高：
  navigationView.widthAnchor.constraint(equalTo: view.widthAnchor),
  navigationView.heightAnchor.constraint(equalTo: view.widthAnchor),
  //        - 别的view的宽高的倍数：
  docImage.widthAnchor.constraint(equalTo: view.heightAnchor, multiplier: 0.285)
  //    2. 设置各个边的位置：
  //        - 与别的view相同
  titleLabel.topAnchor.constraint(equalTo: cancelButton.topAnchor), //顶部和另一view在同一条线上
  cancelButton.leftAnchor.constraint(equalTo: view.leftAnchor), //左侧贴着父view的左侧
  //        - 相对别的view偏移一定距离
  uploadButton.bottomAnchor.constraint(equalTo: view.bottomAnchor, constant: -40.0), //底部位于父view底部上方距离40pt处
  navigationView.rightAnchor.constraint(equalTo: view.rightAnchor, constant: 30.0),
  //    3. 中轴与另一view的中轴相等
  fileNameLabel.centerXAnchor.constraint(equalTo: view.centerXAnchor), //与父view相同即相对父view居中对齐
  ...

])
```

通过描述这样的约束，我们可以让AutoLayout知道我们想要的页面大致是什么样的，如果父view的大小和比例改变，或者某个子view由于各种原因大小改变，也可以在遵守一系列约束的情况下相对自由的调整各个view的大小和位置。

# SafeArea

> 相关文章：
>
> - [ iOS Safe Area](https://medium.com/rosberryapps/ios-safe-area-ca10e919526f)
>
> - [Positioning Content Relative to the Safe Area - Apple Developer](https://developer.apple.com/documentation/uikit/uiview/positioning_content_relative_to_the_safe_area)

### 概念

在iOS的UI开发中有一个概念叫做SafeArea（安全区），它的作用是将容器中的某一部分定义为“安全”的，而其余的部分会被占据，排除在我们的布局之外，让我们的组件不能被放置到上面。

如果这样说还不能理解它的好用之处，那么看看系统自带的安全区配置就明白了：

在Iphone的顶部通常都会存在一条系统默认的顶部栏，显示时间、信号、电量等等，在取消Home键之后的机型中（iPhoneX及之后），底部也有一块可以滑动切屏的区域。通常我们都是不想让我们的视图显示在这些区域的，不然就会造成视图或者点击事件被遮盖的情况。尤其是手机顶部那一块“刘海”，在横屏的时候如果不额外设置，就会遮挡一块区域。安全区就是为了解决这一问题而提供的一种便捷工具，保证我们的视图处于一个可以正常的显示和交互的位置。

> - 灰色区域就是一个安全区，绿色区域就是我们想避开的“不安全区”。
>
> <img src="https://miro.medium.com/max/1400/1*XJro6ujERnKVXIcDVXy6kw.png" alt="411d541bb05c39a8b9917e4ab1166fa5" style="zoom:30%;" /><img src="https://miro.medium.com/max/1400/1*rwWMirhEosAfaBnkWOwBpg.png" style="zoom:30%;" />

### 使用

在实际的开发中，我们可以利用系统为我们提供的默认安全区，也可以自己额外配置安全区。

- safeAreaLayoutGuide

```swift
@available（iOs 11.0, *）
open var safeAreaLayoutGuide: UILayoutGuide { get }
```

我们的view都具有一个自带的`safeAreaLayoutGuide`属性，表示默认的安全区。即上面图中展示的灰色区域。

它的类型是`UILayoutGuide`，代表一个虚拟的view，它自己不会显示任何东西，可以在布局的时候用来占位。我们可以把它理解成“抽象”的，因为它不会显示在我们的view层级上，只是帮助我们来对其他view进行布局。

`safeAreaLayoutGuide`就是系统帮我们创建好的安全区的虚拟view，系统会自动排除掉导航栏等等这些不适合展示视图的部分。当我们在配置自动布局(AutoLayout)的时候，就可以根据需要在视图和安全区的边缘之间建立约束。例如：

```swift
bottomButton.bottomAnchor.constraint(equalTo: view.safeAreaLayoutGuide.bottomAnchor),
```

不过如果我们的view本身处在父view的安全区中，并不触及到系统顶部栏和底部区域这些“不安全区”，那它的默认安全区就是它的整个view了。

- safeAreaInsets

```swift
@available（iOs 11.0, *）
open var safeAreaInsets: UIEdgeInsets { get }
```

`safeAreaInsets`这个属性代表的是安全区的边缘。

我们知道`UIEdgeInsets`这个类型表示一个视图的边缘的位置，包括`top`、`left`、`bottom`、`right`四个方向。那么这个`safeAreaInsets`就是安全区的边缘了。当我们在自己配置布局的时候，也可以利用这些边缘值，例如：

```swift
docImage.frame.size.width = (view.frame.width - view.safeAreaInsets.left - view.safeAreaInsets.right) / 2.0
```

- additionalSafeAreaInsets

```swift
@available（iOs 11.0, *）
open var additionalSafeAreaInsets: UIEdgeInsets
```

我们也可以自己通过`additionalSafeAreaInsets`属性来定义额外的inset。例如，如果我们自己创建了一个顶部或者底部的导航栏，为了不让它遮挡住我们中间主要的视图，我们可以在中间的视图中把导航栏占据的部分设置为安全区，让中间的视图自动避开导航栏遮盖的区域。


