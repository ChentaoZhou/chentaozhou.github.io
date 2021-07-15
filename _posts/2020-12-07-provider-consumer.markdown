---
layout: post
title: Flutter学习笔记 ChangeNotifierProvider and Consumer
image: 3.jpg
date: 2020-12-07 17:20:20 +0200
tags: [Flutter]
categories: Flutter
---
# Flutter学习笔记: ChangeNotifierProvider & Consumer

最近接触的一个需求涉及到了Provider和Consumer的使用，非常方便好用，因此简单的学习并记录一下。

参考阅读：

- [Flutter Provider状态管理-Consumer (autonomousjack)](https://blog.csdn.net/u013894711/article/details/102782366)
- [Flutter中文网：简单的应用状态管理](https://flutter.cn/docs/development/data-and-backend/state-mgmt/simple)

## 简介

笼统地说，Provider在Flutter中的作用是“状态管理”，它使得组件的状态可以被共享、同步，以做到及时的更新。

我们平时想要构建一个可以更新的Widget，都是使用StatefulWidget。因为我们知道Widget类本身是不可变（`immutable`）的，它内部的数据都由final修饰，所以StatefulWidget用另一个继承自State的类来保存一个“状态”，在Flutter构建Widget树时，会获取调用这个State的`build`方法去构建一个Widget。我们可以通过给State的`setState`方法传入闭包来改变这个“状态”，这时Widget也会得到更新，也就是我们可以看到页面马上被改变了。

但是这要求我们总是使用StatefulWidget和State，并且它会造成整个页面的重绘。大量的StatefulWidget的存在会影响应用的性能。Flutter推荐我们使用工具Provider来进行状态的管理。它可以实现对StatelessWidget的更新，也可以指定的重绘部分组件。

## ChangeNotifier

ChangeNotifier是我们定义一个Provider时要去继承的父类。顾名思义，它是一个“变化通知器”，它可以维护一组“监听器”，并且给它们发送通知。其实它就类似一个简单的观察者（Observer）模式。我们可能会用到它的这些方法：

- ```dart
  void notifyListeners(); //通知所有监听器（观察者）
  ```

- ```dart
  void addListener(VoidCallback listener); //手动增加监听器（观察者）
  ```

  - 这里的参数其实就是一个无返回值的函数。当有通知要被传达出来时，这些函数就会被回调。

我们的Provider继承ChangeNotifier之后就拥有了被监听、通知监听者的能力，此外，我们还需要：

- 定义所需属性（即被监听的数据）
- 定义方法来获取和修改这些属性（修改属性后都要手动调用`notifyListener`）

例如下面这个Provider就维护了一个可以被初始化和不断增加的`int`值。

```dart
class MyProvider with ChangeNotifier {
  int get count => _count;
  
  MyProvider({int count}): _count = count;
 
  void increment() {
    _count++;
    notifyListeners();
  }
}
```

这样一来就形成了一个完整的观察者模式，我们的Provider就是那个被观察的数据主体。

那么怎么创建这样的Provider实例呢？我们只需要确定一个需要访问它的Widget，然后在创建这个Widget时多包裹一层，为它添加Provider就可以了。

- 单个Provider：接受一个Provider

```dart
runApp(
    ChangeNotifierProvider(
      create: (context) => myProvider(count: 0),
      child: MyApp(),
    ),
  );
```

> 注意：**不建议像这样在程序入口初始化Provider**，实际项目中要是都在程序入口初始化可能会导致内存急剧增加，除非是共享一些全局的状态，例如app日夜间模式切换，中英文切换等。
> [Flutter Provider状态管理-Consumer (autonomousjack)](https://blog.csdn.net/u013894711/article/details/102782366)

- 多个Provider：接受一个Provider类型的List，可以从多个Provider接收数据

```dart
Widget myView = Builder(
  builder: (context) => MultiProvider(
     providers: [
        ChangeNotifier<MyProvider>(
           create: (_) => MyProvider(count: 0)),
        ...
        // 因为是MultiProvider，我们可以类似上面这样在这里可以构造多个Provider
     ],
     child: ... // 这里继续构建实际的Widget
  );
)
```

在Provider中的数据被更新时，如果我们需要某个函数被回调，需要将函数通过`addListener`注册给Provider。

## Consumer

我们可以直接从Provider中获取值来放在页面上展示，这样做的话，相当于自动的监听了Provider的变化（不需要自己注册Listener），可以做到在Provider中数据被更新时实时更新页面。但是，仅仅如此，**还是会造成整个页面的重绘**，例如在这种情况下：

```dart
Widget build(BuildContext context) {
    //获取Provider
    MyProvider myProvider = Provider.of<MyProvider>(context);
    return GestureDetector(
      child: Text('${myProvider.count}'),
      onTap: () {
        //更新数据
        myProvider.increment();
      }
    );
  }
}
```

我们用`of`方法，利用当前的context，向树的上方寻找并获取到了我们的Provider实例。这时，当Provider中的count被改变时，页面会即使的更新值，但是，实际的操作依旧是**整个Widget的`build`方法被重新调用了**，整个页面都被重绘，而不仅仅是使用了Provider数据的Text。这样一来并不能完全满足我们“局部重绘”的要求。

这时Consumer就出场了。Consumer也是一个Widget，它的参数有：

- context

- builder：Consumer的builder会在Provider发生变化时被重新调用。也就是说，我们只要把**需要被更新**的那部分内容放在Consumer的builder中，那每次数据更新，都只有这部分会重绘了。
- changeNotifier：把对应的**Provider**放在这里。
- child：这部分的Widget是**不会**因builder的重新调用而更新的。因此，如果Consumer下还有大量的子Widget，我们可以把它放在这里，避免每次更新进行重绘。

可以看出，Consumer可以指定一部分内容物受到Provider更新的影响，它的内外的其他部分都可以避免被无意义的重绘了～一般来说，我们尽量将它放在Widget树相对较低的位置，来避免过多Widget的重绘。

例如：

```dart
Widget build(BuildContext context) {
  return Consumer<MyProvider>(
    builder: (context, myProvider, child) => GestureDetector(
      child: Text("${myProvider.count}"),
      onTap: () {
        myProvider.increment();
      },
    ),
  );
}
```

另外，Flutter还为我们提供了`Consumer2`、`Consumer3`一直到`Consumer6`的类型，在我们的组件需要多个不同的Provider中的数据时，允许我们添加2-6个不同的Provider。

## 只访问不监听

> 有的时候你不需要模型中的 **数据** 来改变 UI，但是你可能还是需要访问该数据。

这种情况下我们可能不需要重绘Widget，不需要每次Provider改变都知会我们。这时我们要在获取Provider时将`listen`设置为`false`。

```dart
Provider.of<CartModel>(context, listen: false);
```
