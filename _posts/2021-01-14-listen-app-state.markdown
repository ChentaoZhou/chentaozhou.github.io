---
layout: post
title: 在Flutter页面中监听应用状态变化
image: 8.jpg
date: 2021-01-14 16:36:20 +0200
tags: [Flutter]
categories: Flutter
---

这两天做了一个应用隐私权限的页面，可以显示和修改当前应用对系统的权限，例如相机、相册、地理位置等等。期间发现这样一个情况：在iOS平台上，如果应用正在后台运行，但是我们手动打开系统设置改变了权限，再切回前台，会发现应用已经被自动关闭了，需要重启。但是在安卓平台上，当前的系统版本可以做到在权限发生变化的时候不关闭应用（进程和数据可能发生了变化，但是页面还是保留的）。

那么在这种情况下，为了保证页面上的权限状态可以及时的更新，我们需要在变化发生时，或者是应用切回前台时，重新加载页面的状态。

经过检索，我发现Flutter为我们提供了一个很简单的方式来实现这个要求， 那就是`WidgetsBindingObserver`接口。

### WidgetsBindingObserver

WidgetsBindingObserver其实是一个abstract class，它为我们提供了从Widget层来观察应用的一些状态变化的方法，让我们可以及时得到一些状态变化的信息，并以此来作出反应，例如更新页面State。

这里列举一部分方法：

- `didChangeMetrics()`：在应用的维度变化时（例如手机旋转时）被调用。
- `didChangePlatformBrightness()`：在平台系统亮度模式（黑暗模式等）改变时被调用。
- `didChangeAccessibilityFeatures()`：在系统相关的可访问性（权限）特性被改变时调用。
- `didChangeTextScaleFactor()`：系统字体大小设置发生改变时调用。
- `didHaveMemoryPressure`：当系统的内存低时被调用。

...此外还有监听路由进出的接口，在这里就不一一列举了。但是这里想介绍的主要是下面这个接口：

- `didChangeAppLifecycleState(AppLifecycleState state)`：在应用被切换到后台或者回到前台时被调用。

#### didChangeAppLifecycleState

首先看这个方法的参数`AppLifecycleState`，看起来似乎是对应用生命周期中的状态的枚举。那么我们肯定要先来看看在Flutter中定义的应用生命周期到底有哪几种状态：

```dart
enum AppLifecycleState {
  resumed,
  inactive,
  paused,
  detached,
}
```

看来我们可能需要关注的状态只有四种！看起来非常好理解，简单的说明一下：

- `resumed`：应用在前台，对用户可见，可以对用户的输入产生响应。

- `inactive`：应用虽然在前台，但是处于一个“不活跃”状态，无法接受和响应用户输入。

  - 在iOS系统中，常见的情景包括电话呼入、切换应用的过程中，拉下控制中心等等。
  - 在安卓系统中，则包括电话呼入以及焦点转移的情况，例如有画中画应用、系统弹窗出现等等。

  在这种情况下，很可能接下来在某个时刻应用就会进入`paused`状态了！

- `paused`：应用在后台，对用户不可见，无法响应用户输入。

- `detached`：这个单词的意思是“分离的、独立的、不连接的”，在这种情况下，应用程序依然运行在Flutter引擎上，但是没有和任何视图相连接。会出现这种状态的情况包括Flutter引擎刚刚初始化或者一个view被销毁后，还正在连接新的view，但尚未完成的时候。

这些状态和安卓和iOS系统中的一些页面状态相对应，在早期的版本中，有一些状态只对一个平台有意义，但是目前版本的这四个状态在两个平台上都可以表现出来。虽然它只有四种状态，相对来说比较粗略，但是已经可以做到满足很多场景了。

了解了这些状态之后，再回到这个方法上来。`didChangeAppLifecycleState`会在应用状态发生改变时被调用，同时传入当前的应用状态。那么我们只需要判断这个状态是什么，就可以作出相应的反应了。

#### 使用

现在来看一下使用的过程，以上面所说的`didChangeAppLifecycleState`方法为例，非常简单：

1. 首先要让我们页面的`State`类混入`WidgetsBindingObserver`。

2. 其次重写`initState`方法，在其中将自身作为**观察者**注册给`WidgetBinding`。

3. 还需要重写`dispose`方法，在页面退出时把自己从观察者中移除。

4. 最后重写`didChangeAppLifecycleState`，定义应用状态变化后不同情况下的操作。

假设我们的一个页面"MyPage"如下：

```dart
class MyPage extends StatefulWidget {
  @override
  _MyPageState createState() => _MyPageState();
}

class _MyPageState extends State<MyPage> with WidgetsBindingObserver { // 混入接口
  @override
  void initState() {
    WidgetsBinding.instance.addObserver(this); // 添加观察者
    super.initState();
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this); // 移除观察者
    super.dispose();
  }
  
  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.inactive: 
        // ... ...
        break;
      case AppLifecycleState.resumed: 
        // ... ...
        break;
      case AppLifecycleState.paused: 
        // ... ...
        break;
      case AppLifecycleState.detached: 
        // ... ...
        break;
    }
    // 如果只想对某一种状态作出反应就不用写这么多，例如在页面回到resume的时候刷新页面：
    if (state == AppLifecycleState.resumed) {
      setState(() {
        // ... ...
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    // ... ...
  }
}
```

其他的方法也都同理，是不是非常好用呢！

