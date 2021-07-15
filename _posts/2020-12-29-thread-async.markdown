---
layout: post
title: Dart的单线程模型与异步编程(Future、Stream)
image: 6.jpg
date: 2020-12-29 16:36:20 +0200
tags: [Flutter]
categories: Flutter
---
**Dart是一个默认单线程的语言，它同样也支持异步编程。**
我们知道例如Java在内的很多语言是可以通过多线程来实现异步操作的，但是Dart是在单线程中实现异步编程。对JS了解比较多的同学应该清楚这是怎么做的，因为Dart的单线程模型其实就是借鉴了JavaScript的单线程模式，它们实现异步的方案是使用**事件队列和事件循环**。

***

首先我们需要明确，这里的“单线程”并不是一个百分百准确的描述。需要注意的是：
1. 这里的“线程”指Dart中的isolate，它和线程的概念并不完全一致。

2. 单线程是相对于多线程的语言而言的，能执行我们的Dart代码的只有一个线程。但是要知道整个Flutter系统可以分成三层，Dart语言只是它的第一层（framework层），在底层还有操作系统提供的线程去帮我们完成例如I/O、GPU绘制、渲染、Timer等等操作。

    所以说，在Dart层中所做的事其实是有限的，对于网络操作、I/O等等可能耗时的操作，Dart只需要在获取到结果时可以收到通知并且执行后续操作就可以了。那么只要有能力避免线程被等待所阻塞，就可以用单线程来避免多线程存在的可能会很复杂的同步问题。这时候就要让事件队列和事件循环出场了。
    
## 事件循环和事件队列

单线程模型使得我们的任务只能由一个线程按照顺序依次来执行。在Dart中，这个运行main函数的线程被称为main isolate。这个所谓的isolate其实就是类似与线程thread的存在，但是又略有区别，这个我们后面再进行讨论。

在dart中，一个这样的isolate就会包含包含一个事件循环和两个事件队列：

- 事件循环：线程会从某个队列中取出一件事情来做，做完后再次去获取。
- 事件队列：需要做的事情会在队列中等待线程来取用，不同的事情根据优先级被放在不同的队列中。
  1. Microtask queue 微任务队列（高优先级）
  2. Event queue 事件队列（低优先级）

![enter image description here](http://km.oa.com/files/photos/pictures/202012/1609230397_15_w471_h506.png)

从这张图片可以很清晰的看出这个循环的过程和两个队列的优先级关系了：

- 如果微任务队列不为空，线程会不断取其中的任务来执行。
- 如果微任务队列为空，才会去执行一个事件队列中的任务，然后再回去检查微任务队列是否为空。

- 两者都为空时，退出程序。

那么要进行异步操作就很简单了：我们只需要创建一个任务，把它加到某个事件队列中去，就可以让我们的线程在**之后的某个时间再去执行它**。

## 创建异步任务

**创建microtask微任务**

```dart
scheduleMicrotask(() => print('This is a microtask'));

Future.microtask(() => print('This is also a microtask'));
```

微任务队列的优先级比事件队列高，所以微任务一定会在所有事件任务之前完成。

**创建事件队列任务**

我们可以通过创建`Future`对象来创建一个事件队列上的任务，这是Dart为我们提供的一个包装。

### Future的使用

顾名思义，Future就是未来，`Future`类型代表的就是一个会在未来某个时刻得到的返回值（也可以没有返回值，只是使一些操作在未来特定的时候执行）。我们可以在创建时给它传递一个闭包，其中的操作就作为一个任务被放进事件队列中去了。

```dart
main() {
    print('main start');
    var future1 = Future(() => print('future1 running')); 
    Future(() => print('future2 running')); // 匿名的Future
    print('main end');
}
// main start
// main end
// future1 running
// future2 running
```

在上面的例子中，我们封装了一个Future，它没有返回值，仅仅包含一个`print`操作。当`Future`被创建时，这个操作被放在了事件队列中，当`main`函数执行完毕之后才会去依次执行。

#### 链式调用

当一个`Future`被执行完毕之后，有时候我们希望可以马上知道，或者马上回调某些函数。`Future`的`.then()`链式调用可以达到这样的效果。在有返回值的情况下，我们可以在`then`中取用这个返回值。

```dart
main() {
    Future(() => print("future1 running"))
      .then((_) => print("future1 is done"))
      .then((_) => print("second then"));
    Future(() => print("future2 running"));
}
// future1 running
// future1 is done
// second then
// future2 running
```

从上面的例子中，我们发现，当第一个`Future`执行之后，它的两个`then`接着被执行了，然后才进入下一个Future。可以知道，`Future`和它的`then`是在**同一个事件循环内**完成的。在一个事件`then`执行完成前，线程不会再去队列中取事件。

但要注意的是，有一种情况会将`then`打断：

```dart
main() {
    Future(() => print("future1 running"))
        .then((_) => Future(() => print("This is a nested future")))
        .then((_) => print("second then"));
    Future(() => print("future2 running"));
}
// future1 running
// future2 running
// This is a nested future
// second then
```

这一次，第一个`Future`的`then`竟然没有先于第二个`Future`执行，这是因为在第一个`then`执行的时候，我们又创建了一个`Future`，这个`Future`会被放在事件队列的最后，同时它之后的那个`then`就也被放到最后去执行了。

#### 其他操作

- **延迟任务**

`Future`还有一个`.delayed()`方法让我们可以使得这个任务在一段时间后再被添加到队列中：

```dart
main() {
    Future.delayed(const Duration(seconds:1), () {
        print("Hello1");
    });
    Future(() => print("Hello2"));
}
// Hello2
// Hello1
```

上面的第一个`Future`会等待1秒才被添加到队列中，所以它会被放在第二个Future之后，自然也会在其后才执行。

- **直接包装一个值**

`.value`方法可以直接包装一个值到`Future`中，可以在`then`中取用，但是`then`中的操作依然会作为任务被放入事件队列，等待`main`结束后执行。

```dart
main() {
  print("main start");
  Future.value("Hello").then((value) => print(value));
  print("main end");
}
// main start
// main end
// Hello
```

### <异步函数：async和await

我们知道，异步函数会在操作还没有完成的时候就返回。那么在Dart中，很好理解，我们可以使函数返回一个`Future`对象，`Future`中的操作的返回值才是异步函数真实的返回值。拿到返回的`Future`之后，我们就可以通过使用`.then()`来定义操作真正执行之后需要进行的处理或者回调。

```dart
Future<String> fetchContent() =>
    Future<String>.delayed(Duration(seconds:3), () => "Hello");

void asyncHello() async{
    print("asyncHello start");
    String str = await fetchContent();
    print(str + "!");
}

main() {
    asyncHello();
    print("main end");
}
```

上面这个例子中，我们定义了一个返回一个`Future<String>`的异步函数`fetchContent`。它返回了一个需要等待三秒才会添加进队列中的`Future<String>`，它封装的真实返回值是`"Hello"。`

然后，我们又定义了一个函数`asyncHello`，这个函数有一个关键词`async`。

- **async/await:** `async`的意思是：这个函数中有一个耗时操作，我们需要等待这个耗时操作完成。同时，必须要用`await`关键词来标记这个耗时操作。那这样一来，不是又造成了等待吗？是的，所以说，这也是一个异步函数，需要进行异步的等待。当运行到`await`时，`await`语句和它的上下文都会被放入事件队列中，等到我们`await`所等待的耗时操作得到了结果，再来继续运行后面的操作。

那么我们就可以来梳理一下上面的例子开始运行时会发生什么：

1. `main`函数运行，调用了`asyncHello`。
2. `asyncHello`开始执行，输出`asyncHello start`
3. 运行到`await`，上下文放入事件队列进行异步等待
4. `main`函数继续运行，输出`main end`，结束
5. 开始执行事件队列，运行`await`的`fetchContent`调用，三秒后事件执行，得到返回值，即含有`Hello`的`Future`
6. `.then`运行，输出`Helloworld`
7. 继续运行后续操作，输出`!`

```
flutter: asyncHello start
flutter: main end
flutter: Helloworld
flutter: !
```

这里要注意的一点是，`main end`并不会等待`asyncHello`完全执行结束再执行。也就是说`async`和`await`的效果不会传递到调用者，也就是`main`函数中。我们存入事件队列的操作和上下文，仅仅局限于这个异步函数内部。如果想让`main`也进行等待，则需要将`main`也用`async`标记，并且将`asyncHello`调用标记为`await`。

## Dart多线程机制

虽然说Dart是默认单线程的，但是它其实也提供了多线程的机制。刚才说过，Dart中的主线程是main isolate，我们也可以创建其他的isolate，以达到多线程的效果。在我们实际进行Flutter项目的开发中，我们可能会需要一些耗时操作并发进行，使其不影响需要及时作出的相应，这时候我们就要创建新的isolate来完成这些操作了。

### Isolate

isolate的底层其实依然是我们操作系统提供的OS Thread，但是Dart基于它抽象了Dart VM Thread，又接着抽象出了isolate的类型供我们使用。isolate在文档中的定义是：一个孤立的Dart执行上下文。可以看出，isolate和我们一般说的线程还是有一些细微的区别的。其关键就在于”孤立“。不同的isolate之间是不会共享资源的。每个isolate都有自己的事件循环和事件队列，它们之间的通信只能靠消息来完成。

创建一个isolate的方法如下：

```dart
doSth(msg) => print(msg);

main() {
  Isolate.spawn(doSth, "Hi");
  ...
}
```

可以看出，创建`Isolate`的函数`.spawn`需要的参数包括一个入口函数，以及函数的参数。

但是在实际的使用中，我们可能还需要在isolate之间进行通信，例如让某个isolate进行运算，然后告知main isolate运算的结果。这样的通信实际上可以通过管道来实现。

- 在接收方创建管道`port = ReceivePort()`，将其包含的发送入口`port.sendPort`传递给发送方
- 发送方通过发送入口发送消息`port.send()`
- 接收方可以选择等待特定的消息`final message = await port.first`
  - 或者监听消息`port.listen((data) {...})`

这样的一条管道只能进行单向的通信。如果我们需要双向的通信，那么还需要并发isolate向主isolate回传一个管道。为了方便起见，我们可以引入`package:flutter/foundation.dart`，利用Flutter提供给我们的`compute()`函数来实现并发运算。我们只需要提供计算用的回调函数和参数作为`compute`的参数，就可以创建出并发的isolate来执行这个函数。

```dart
// 源码中的定义：
typedef ComputeCallback<Q, R> = FutureOr<R> Function(Q message);
typedef _ComputeImpl = Future<R> Function<Q, R>(ComputeCallback<Q, R> callback, Q message, { String? debugLabel });
// compute函数就是这里定义的_ComputeImpl类型
```

另外，我们也可以自己去封装一个简单方便的库来使用isolate。

可以看看腾讯文档项目中的`IsolateRunner`：https://git.code.oa.com/tencent-doc/TencentDocsApp/tree/master/packages/IsolateRunner （willisdai）

## Stream

一个写的很好的介绍：[Dart | 什么是Stream (by Vadaski)](https://juejin.cn/post/6844903686737494023)

Stream也是Dart为异步提供支持的核心API。

顾名思义，Stream就是一个”流“。相比Future这种单个的异步数据，它是**一系列有序的异步数据形成的流**。我们可以往这个流中放入很多个东西，然后这些东西可以被处理加工（可能是耗时操作），最后我们会得到相应的结果。我们可以在出口处来监听Stream的输出结果，具体的方法会在后面介绍。

### Stream的种类

首先看一下两种不同的Stream：

- **单订阅流 Single-subscription Stream**

非常”专一“的一种流，只能有一个监听器。它在有收听者之前不会生成事件，并且在取消收听时它会停止发送事件。如果它的监听者取消监听了，也不能再为它增加别的监听者了。

- **广播流 Broadcast Stream**

可以有任意数量的监听者，并且无论是否有收听者，他都能产生事件。所以中途进来的收听者是无法收到之前的消息的。

### 使用Stream

Stream有很多种创建方式：

- #### **构造器**

  - `Stream.empty()`创建一个空的广播流
  - `Stream.fromFuture(Future<T> future)`从Future创建一个单订阅流
  - `Stream.fromFutures(Iterable<Future<T>> futures)`从一组Future中创建单订阅流
  - `Stream.fromIterable([Iterable]<T> elements)`创建一个获取一个集合中的数据的单订阅
  - `Stream.periodic(Duration period, [int Function(int) computation])`创建一个按周期重复发送事件的流
  - ...更多构造器可以看官网文档：[Stream<T> class](https://api.dart.dev/stable/2.10.4/dart-async/Stream-class.html)

- #### 使用`async*`、`yeild`

  不同于`sync`/`async`，带星号的版本`sync*`/`async*`表示的是**多元素**的同步/异步。`sync*`关键词表示一个函数必须返回一个`Iterable<T>`对象，而`async*`表示一个函数必须返回一个`Stream`对象。

  ```dart
  Stream<int> countStream(int to) async* {
    for (int i = 1; i <= to; i++) {
      yield i;
    }
  }
  ```

  上面这个例子中，我们用循环将从1到参数to之间的所有数字都放进了一个流中。注意这里还有一个`yeild`关键词，不同于`return`，它不会使函数终止，只是将一个值放进输出流。

- #### **StreamController**

  `dart:async`库中提供的StreamController类表示一个包含一个Stream的流控制器，它为我们提供处理流的一些方便的功能。

  - 创建一个处理特定类型数据的流（不标注泛型类型则可以发送任何类型的数据，另外，还可以选择给构造器传入回调函数`onListen，onPause，onResume`）。

    ```dart
    StreamController<int> numController = StreamController();
    ```

  - 发送数据`.sink.add()`（`sink`属性会获取到一个StreamSink对象，我们可以把它当成流之上的一个“水池”，或者说是数据接收器，用`add`方法将东西放进去之后就会"流"进这个流中）

    ```dart
    numController.sink.add(123);
    ```

  - 通过监听获取数据，传入回调函数，打印获取到的数据。（这里的`stream`属性就是获取了Controller所管理的这个Stream对象，然后我们调用了它的`listen`方法。）

    ```dart
    StreamSubscription subscription = numController.stream.listen((data)=>print("$data"));
    ```

  这里要关注一下Stream的`listen`这个函数，就是我们监听流的方法。

#### listen

文档中的源码定义：

```dart
StreamSubscription<int> listen(void Function(int) onData, {Function onError, void Function() onDone, bool cancelOnError})
```

可以看到，`listen`需要的参数都是回调函数。但是必填的只有`onData`，也就是**收到数据时触发**的函数。此外还可以定义收到Error时触发、结束时触发的函数，最后的`cancelOnError`就决定是否在遇到Error时取消监听。这个方法返回一个StreamSubscription类的对象，代表一个订阅。通过这个对象我们可以修改订阅的那些回调函数，还可以暂停、重启、取消订阅。

#### where

`where`方法可以让我们对流中的事件做筛选，然后就可以对不同的事件作出不同的反应。例

```dart
stream.where((event){...})
```

#### take

`take`方法可以控制一个流最多能传多少个东西。传完相应的次数之后流就会关闭。

```dart
stream.take(4);
```

#### transform

`transform`方法需要我们提供一个`StreamTransformer<S,T>`（流转换器）作为参数。它的作用是**接收一个流的信息，然后可以对这个流中的东西做某种处理，再返回一条新的流。** 其中的泛型类型`S`代表输入流的输入类型，`T`代表输出流的输出类型。

可以看[什么是Stream？](https://juejin.cn/post/6844903686737494023)中的这个例子：

> ```dart
> StreamController<int> controller = StreamController<int>();
> 
> final transformer = StreamTransformer<int,String>.fromHandlers(
>  handleData:(value, sink){
>     if(value==100){
>    sink.add("你猜对了");
>  }
>     else{ sink.addError('还没猜中，再试一次吧');
>  }
> });
> 
> controller.stream
>          .transform(transformer)
>          .listen(
>              (data) => print(data),
>              onError:(err) => print(err));
>  
>  controller.sink.add(23);
>  controller.sink.add(100);
> ```
>
> 输出：
>
> 还没猜中，再试一次吧
>
> 你猜对了

上面这个例子中，我们定义了一个`StreamTransformer`，用它的`fromHandlers`方法为它定义了一个处理数据的函数。其中的参数`value`代表输入流中的数据，`sink`代表新的输出流的输入入口池。我们可以在其中对输入流的数据做处理，输出对应的数据到新的流中。在这个例子中输入流的类型是`int`，输出流是`String`。然后我们对这个`int`类型的流调用`transform`来将这个Transformer应用在它身上。这时返回的值就已经是一个`String`类型的新流了。我们再链式的调用`listen`，这时我们监听的，其实就是这个新的流了，因此回调函数中打印出来的就是我们在Transformer中定义的`String`类型的数据了：`还没猜中，再试一次吧/你猜对了`。

## 异步更新UI

我们在Flutter应用的开发中，使用一些异步数据来更新UI可能会是非常常见的需求。例如需要从互联网上获取数据来展示，或是需要用Stream来持续的展示某个事件的进度。为了方便满足这个要求，Flutter为我们提供了两个Widget类：**FutureBuilder**和**StreamBuilder**。

这两个组件的使用在文档中有非常详细的例子：[异步UI更新（FutureBuilder、StreamBuilder）](https://book.flutterchina.club/chapter7/futurebuilder_and_streambuilder.html)

简单地说，FutureBuilder可以等待一个Future的执行并且根据结果更新UI；StreamBuilder可以监听一个流并且根据流中获取的数据持续的来更新UI。
在理解了Future和Stream的用法之后，相信也可以很好地理解这两个组件的用法了。

