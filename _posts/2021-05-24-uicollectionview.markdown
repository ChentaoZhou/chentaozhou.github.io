---
layout: post
title: iOS - UICollectionView 使用学习
image: 13.jpg
date: 2021-05-24 16:36:20 +0200
tags: [iOS, Swift]
categories: iOS
---
`UICollectionView`是iOS开发中比较常用的一种组件。它通常用于构建栅格布局（Grid Layout）的视图。

***

### 简介

类似这样：

![rybwD](image/rybwD.png)

或者这样：

![CollectionView](image/CollectionView.jpg)

我们将栅格中的一个单元格称为一个`cell`。这些`cell`是我们的核心视图，也是我们真正想要展示的数据。我们可以将这些`cell`分成不同的组，即`section`。每个`section`都可以有自己的`header`和`footer`，即顶部和底部的文字或组件，整个UICollectionView也可以拥有全局的`header`和`footer`。这些视图被称为追加视图（Supplementary Views）。另外还可以给它配置额外的装饰视图（Decoration Views），例如给`cell`增加背景视图。这些内容一并组成了我们的`UICollectionView`。

它类似于另一个常用组件`UITableView`，即列表式布局的视图：

![TableView](image/TableView.png)

可以说`UICollectionView`是多列版本的`UITableView`。和`UITableView`就是我们最常见的列表布局，iOS的设置页面就是一个最典型的例子，它也可以将`cell`分为多个`section`，并且为他们增加`footer`和`header`。

当`UICollectionView`和`UITableView`中的`cell`数量多起来超出页面能展示的范围时，它是可以进行滑动的，例如设置页面可上下滑动的效果。这是因为它们二者都继承自`UIScrollView`：

![enter image description here](image/ScrollView.jpg)

顾名思义，`UIScrollView`就是可滚动的视图，也是非常常用的视图之一。我们可以给它添加一个子视图作为内容，然后`UIScrollView`本身就像一个窗口一样可以在通过滑动来在屏幕上展示子视图的一部分。`UITableView`和`UICollectionView`的列表滑动的效果就是基于这样的能力实现的。

### 使用

#### 数据源

首先，需要知道我们构建栅格的数据该放在哪里。`UICollectionView`有一个类型为`UICollectionViewDataSource`的实例属性`dataSource`，它就是我们的数据源。我们需要自己去实现`UICollectionViewDataSource`协议，实现它其中提供数据的方法。常见的做法是直接用`UICollectionView`所在的上层类(可能是ViewController或父View的类)实现协议，将`self`作为`dataSource`。

`UICollectionViewDataSource`协议中可以实现的方法包括：获取Item和Section的数量、获取显示的具体视图（`cell`、`header`或`footer`）、定义Item是否可以移动等。其中必须实现的方法只有获取一个section中`cell`的数量，以及获取`cell`本身。

```swift
// 返回Section数量
optional func numberOfSectionsInCollectionView(collectionView: UICollectionView) -> Int

// （必须实现）返回每一section有多少个cell
func collectionView(collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int

// （必须实现）获取cell视图，参数cellForItemAtIndexPath会给出被配置的cell的位置(第几个section的第几个cell)
func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: NSIndexPath) -> UICollectionViewCell

// 获取header或footer视图
optional func collectionView(_ collectionView: UICollectionView, viewForSupplementaryElementOfKind kind: String, atIndexPath indexPath: NSIndexPath) -> UICollectionReusableView

// 能否移动cell
optional func collectionView(_ collectionView: UICollectionView, canMoveItemAtIndexPath indexPath: NSIndexPath) -> Bool

// MARK: 结束移动cell
collectionView(_:moveItemAtIndexPath:toIndexPath:)
```

可以看出，除了第一个`numberOfSectionsInCollectionView`方法之外，其他方法具有相同的方法名和第一个参数，即`UICollectionView`实例本身，最后一个参数名才真正点明了这个方法的作用，也是我们需要的关键信息。

这些方法中最关键的获取`cell`视图的方法的返回值类型是`UICollectionViewCell`，这个类型代表的就是在`UICollectionView`中的一个`cell`，也就是item。那么我们要怎样创建一个`cell`呢？

##### 注册cell

首先，在`UICollectionView`被初始化后，我们就要通过`register`方法注册一个具体的`cell`的类型和一个标识符。

```swift
func register(_ cellClass: AnyClass?, forCellWithReuseIdentifier identifier: String)
```

一般来说`cellClass`可以直接传入`UICollectionViewCell.self`。但是如果想要更方便的统一自定义`cell`的数据、视图和布局等等，也可以自己写一个`UICollectionViewCell`的子类（不需要额外实现其他方法）并传入这个类型。

至于第二个参数`identifier`的用途，就要先提到`cell`重用的机制了。

##### 重用cell

`UICollectionViewCell`是`UICollectionReusableView`的子类。从名字可以看出，它是一个"可复用"的视图。有时候，我们的`UICollectionView`中的`cell`会非常多，一个屏幕的空间无法显示完全，需要我们滑动才能显示出更多。在这种情况下，如果我们对每个`cell`都使用新的视图对象，那么这些视图会很占用空间。如果在`cell`滚动出可见范围时把它们删除，滚入时再重新创建，也需要不断地初始化和回收对象。`UICollectionView`的做法是将`cell`放入一个"重用队列"中，后续则可以在需要的时候从中取用。如果队列中没有我们需要的`cell`，再重新创建。

`UICollectionView`有一个实例方法叫做`dequeueReusableCell`，即从复用队列中取出`cell`：

```swift
func dequeueReusableCell(withIdentifier identifier: String) -> UITableViewCell?
```

我们需要通过注册时设置的特定的`identifier`，在复用队列中找到当前`UICollectionView`的`cell`，将它取出来重新取出来使用。

这里需要注意的是，当`cell`被从重用对列中取出时，它的各种属性、子视图都依旧保持着上次被展示出来时的状态。因此，如果我们仅仅在服用时向`cell`添加子视图，当我们反复滚动页面时，`cell`上的子视图就会不断增加堆叠起来。类似这样：

![堆叠.png](image/堆叠.png)

如果我们想用它展示全新的内容，就必须要清除之前的内容。一种做法是直接清除其中的所有旧子视图：

```swift
let myCell = collectionView.dequeueReusableCell(withReuseIdentifier: "MyCell", for: indexPath)
myCell.subviews.forEach({ $0.removeFromSuperview() })
myCell.addSubview(newSubview)
return myCell
```

另外我们也可以如前文所说，自己定义一个`UICollectionViewCell`的子类，将关键的子View作为属性存放在其中，然后在重用时更新这个子View。

#### 布局

对于只有一列的`UITableView`的时候，布局是默认的。但是在使用`UICollectionView`的时候，我们需要使用`UICollectionViewFlowLayout`来自定义视图的一些布局细节。基本的布局方式依旧是固定的，即流式布局（flow layout），也就是在一个方向上固定距离，另一个方向可以滚动，在这样的容器中排列我们的`cell`们，（如果上下滚动，就一排一排填，如果左右滚动，就一列一列填）。

##### 固定的布局属性

需要我们去自定义的细节则包括：滚动方向（默认为纵向）、`cell`的大小、`cell`的横向纵向最小间距、Header和Footer的大小等等。如果这些数据是固定的，我们可以直接通过创建`UICollectionViewFlowLayout`实例，设定它对应的属性，再在构建`UICollectionView`时传入就可以了，例如：

```swift
let layout: UICollectionViewFlowLayout = UICollectionViewFlowLayout();
// cell的大小
layout.itemSize = CGSize(width: 60, height: 60)
// 最小行间距，默认是0
layout.minimumLineSpacing = 5;
// 最小cell间距，默认是10
layout.minimumInteritemSpacing = 5;
// section内间距，默认是 UIEdgeInsetsMake(0, 0, 0, 0)
layout.sectionInset = UIEdgeInsets(top: 10, left: 10, bottom: 10, right: 10);
// 布局方向，默认是UICollectionView.ScrollDirection.vertical
layout.scrollDirection = UICollectionView.ScrollDirection.horizontal
// 使用传入layout的构造器
let contentCollectionView: UICollectionView = UICollectionView(frame: CGRect(), collectionViewLayout: layout);

... ...
```

注意`minimumInteritemSpacing`和`minimumLineSpacing`这两个属性，它们叫做单个`cell`之间的间距或每行(横向滑动时是每列)间距的"最小值"。![间距](image/间距.png)

首先，在排列`cell`的时候，系统一开始会以这个最小的间距去排，但是如果一行已经装不下下一个`cell`，但是又存在剩余空间的时候，这些空间就会平均分配给这一行的`cell`的间距。在所以实际的间距很可能是比这个最小值大的。对于行/列间距来说，如果每个`cell`的高度/宽度不一致时，只有最高/宽的`cell`会适用最小的间距。

说到高度/宽度不一致的`cell`，这在上面的例子中是不存在的，刚才我们是用布局实例的`itemSize`配置了`cell`的大小，它是一个固定的数值。但如果我们希望一些cell的大小是不同的，或一些cell的间距和其他的不同，就需要换一种方式来进行配置了。

##### 为cell/section配置不同的布局

类似于`dataSource`，`UICollectionView`还有一个类型为`UICollectionViewDelegate`的`delegate`属性，意为"代理"。`UICollectionViewDelegate`协议用于管理用户和`UICollectionView`的交互，此外，它有一个扩展协议，名为`UICollectionViewDelegateFlowLayout`，它额外提供了更详细的自定义布局能力。和配置`UICollectionViewDataSource`一样，我们需要实现这个协议，实现其中我们所需要的方法，然后将实现了这个协议的的类的对象赋给`delegate`属性，让它成为了我们配置视图属性的"代理"。

`UICollectionViewDelegateFlowLayout`包含一些可以为不同`cell`或`section`来定义独立的布局的方法。例如：

```swift
// 各个cell的大小，参数sizeForItemAt会给出cell在collectionView中的坐标(sectionInIndex, index)
func collectionView(UICollectionView, layout: UICollectionViewLayout, sizeForItemAt: IndexPath) -> CGSize

// 各个section中cell之间的间距，参数minimumInteritemSpacingForSectionAt会给出是第几个section
func collectionView(UICollectionView, layout: UICollectionViewLayout, minimumInteritemSpacingForSectionAt: Int) -> CGFloat

// 各个section中的行/列间距
func collectionView(UICollectionView, layout: UICollectionViewLayout, minimumLineSpacingForSectionAt: Int) -> CGFloat

// 各个section中的cell的内间距
func collectionView(UICollectionView, layout: UICollectionViewLayout, insetForSectionAt: Int) -> UIEdgeInsets

// ... 此外还包括一些配置footer, header布局的方法等等
```

这些方法`UICollectionViewDataSource`和定义的那些方法一样具有相同的方法名，另外它的第二个参数总是会传入我们先前初始化`UICollectionView`时传入的那个`UICollectionViewLayout`布局实例。

如果我们想给每个`cell`以不同的大小，那么则需要实现上述第一个方法，根据`sizeForItemAt`参数返回不同的结果。系统会为每个`cell`调用一次这个方法来确定它的大小。

当我们实现了这些方法之后，先前在`UICollectionViewFlowLayout`中配置的那些旧的属性就自然不会生效了，如果不想修改个别`cell`或`section`的布局，就在方法里为它们直接返回参数中的旧`layout`的属性。

#### 交互

如上一节所说，`UICollectionViewDelegate`协议用于管理用户和`UICollectionView`的交互。其中可以实现的方法非常多，功能上包括管理`cell`的手势交互、管理`cell`的高亮、管理重用视图的添加和移除（滚入和滚出视图的前后）、处理布局的变化、是否允许显示菜单等等。

一般情况下我们都希望`UICollectionView`中的视图可以响应用户的点击事件，因此较常用的方法包括：

```swift
// 在方法中响应对cell的点击，indexPath参数即被点击的cell的坐标
func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) -> Void
```

这个协议中的所有方法都是可选的，可以根据需要去实现。在iOS 11或以上还可以通过实现`UICollectionViewDropDelegate`协议来实现用手势拖拽`cell`重新排序的功能。

---
对以上这些协议的实现基本勾勒出了`UICollectionView`的完整表现。

