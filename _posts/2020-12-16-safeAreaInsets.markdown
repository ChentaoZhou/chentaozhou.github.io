---
layout: post
title: 在viewDidLoad中获取safeAreaInsets都为0的问题
image: 2.jpg
date: 2020-12-16 20:01:00 +0200
tags: [iOS]
categories: iOS
---
A problem of getting 0 from safeAreaInsets in viewDidLoad.
前两天在用AutoLayout的时候遇到一个问题。在viewDidLoad方法中设置约束的时候，想取`view.safeAreaInset.bottom`来将页面底部的按钮布局到安全区中，结果这个值取出来为0。打断点测试了一下，又打了日志，发现在这里取到的`safeAreaInset`是(0, 0, 0, 0)。

***
> 关于`safeAreaInsets`是什么：[iOS开发：AutoLayout与Autoresizing、SafeArea安全区](http://km.oa.com/articles/show/483729?ts=1608117022)

查询了一下，找到了这个：[stackoverflow - safeAreaInsets in UIView is 0 on an iPhone X](https://stackoverflow.com/questions/47032855/safeareainsets-in-uiview-is-0-on-an-iphone-x)。

其他中文结果基本都是对这个问题的蹩脚翻译。内容大致意思是：**`safeAreaInset`被设置的时间要晚于viewDidLoad的时间。**

首先看一下viewController的生命周期：

- loadView：加载view
- viewDidLoad：view加载完毕
- viewWillAppear：控制器的view将要显示
- viewWillLayoutSubviews：控制器的view将要布局子控件
- viewDidLayoutSubviews：控制器的view布局子控件完成
  - 以上这两个可能调用多次
- viewDidAppear:控制器的view完全显示
- viewWillDisappear：控制器的view即将消失的时候
- viewDidDisappear：控制器的view完全消失的时候

我在后续的部分方法中取了一下`safeAreaInsets`的值并且用log打出来，看看这个值究竟是什么时候能获取到。另外还调用了一下**viewSafeAreaInsetsDidChange**方法。这个方法会在`safeAreaInsets`变化时调用，也就是当insets被设置上去的时候，这个方法是必然会被调用的。

```swift
    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        let test = view.safeAreaInsets.bottom
        NSLog("viewDidLayoutSubviews - test: %f", test)
    }
    override func viewDidAppear(_ animated: Bool) {
        let test = view.safeAreaInsets.bottom
        NSLog("viewDidAppear - test: %f", test)
    }
    override func viewSafeAreaInsetsDidChange() {
        let test = view.safeAreaInsets.bottom
        NSLog("viewSafeAreaInsetsDidChange - test: %f", test)
    }
```

得到的结果如下：

```
viewDidLayoutSubviews - test: 0.000000
viewDidLayoutSubviews - test: 0.000000
viewDidLayoutSubviews - test: 0.000000
viewDidLayoutSubviews - test: 0.000000
viewDidLayoutSubviews - test: 0.000000
viewSafeAreaInsetsDidChange - test: 34.000000
viewDidLayoutSubviews - test: 34.000000
viewDidAppear - test: 34.000000
```

可以看出，viewSafeAreaInsetsDidChange是在viewDidLayoutSubviews多次调用之间被调用了，也就是这时这个值被设置成了正确的值，不再为0。

查询了一下官方文档[AppleDeveloper - safeAreaInsets](https://developer.apple.com/documentation/uikit/uiview/2891103-safeareainsets)，看到这样一段话：

> If the view is not currently installed in a view hierarchy, or is not yet visible onscreen, the edge insets in this property are `0`.
>
> 如果view现在还没有被安装在view层级结构中，或者尚未在屏幕上显示，edge insets的属性为`0`。

这个故事告诉我们，遇到问题记得先查文档...

很显然，我们的view只有显示出来之后，insets的值才会被设定。在此之前获取到的值只能是0。那么我们在这之前取值并且拿这个值去配置布局，结果也只会是0，即使后期insets改变，也不会改变我们曾经取到的值了。

因此，如果我们在设计页面的时候需要利用这个值，要么就重写方法`viewSafeAreaInsetsDidChange`，在Insets改变时再取出来使用，要么就是在`viewDidAppear`中再去使用。

如果想在此之前（例如在`viewDidLoad`中）就根据安全区配置布局，要使用`safeAreaLayoutGuide`的Anchor来配置约束，这样实际的布局效果可以在安全区被设置出来时动态适配。例如：

```swift
NSLayoutConstraint.activate([
  // uploadButton的下方边缘紧贴安全区的下方边缘
  uploadButton.bottomAnchor.constraint(equalTo: view.safeAreaLayoutGuide.bottomAnchor),
  ...
]) 
```
