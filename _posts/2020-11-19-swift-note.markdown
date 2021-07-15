---
layout: post
title: Swift学习笔记 基本语法和概念
image: 4.jpg
date: 2020-11-19 20:01:00 +0200
tags: [iOS, Swift]
categories: iOS
---
为了入门iOS开发，导师给我推荐了一个斯坦福2020春季发布的网课作为Swift语言的学习材料。链接：[CS193p - Developing apps for iOS](https://cs193p.sites.stanford.edu)
目前我已经看到了第三周的课程。这个课程的内容主要分为两个部分，一部分是讲解一些基本的语法和概念，另一部分是手把手的编写一个翻牌配对游戏Demo。虽然是英文授课，但是老师的语音非常清晰，听起来难度不大，讲的也很生动，非常适合Swift入门，有类似需求的同学都可以看一看。
对一门语言熟悉起来最好的方式就是自己写代码，但是最基本的还是要熟悉语法和规则，否则读写代码的效率都会非常低。**笔记仅仅包含一些基本概念，作为记录，帮助理解和记忆，实际的学习还是以Demo为主。**

## 一、MVVM
> MVVM是构建"可交互用户界面"的一种代码结构设计。

学过MVC的同学应该都很容易理解这类模型的概念，课程课件中展示的MVVM示意图是这样的：
![enter image description here](http://km.oa.com/files/photos/pictures/202011/1605787419_66_w2278_h1316.png)

简单的说就是：
#### 1. View -> Model
- View持有ViewModel。当用户在View上进行操作时，View会调用ViewModel中定义的一系列"Intent（意图）"函数。
- ViewModel持有Model。在ViewModel的Intent函数中，ViewModel会调用Model中的函数来进行数据的修改。

#### 2. Model -> View
- ViewModel察觉Model的变化，知会View，View改变绘图

第一部分的持有和调用关系很好理解，但是第二部分仅仅这么说可能有点抽象，但是在学习Demo的过程中就会发现，在Swift中这个过程是这样实现的：
1. 让ViewModel实现协议`ObservableObject`（协议类似于Java中的Interface，后面会解释）
```swift
class ViewModelClass: ObservableObject {...}
```
实现了这个协议就自动拥有了这样一个属性（不用自己写在里面）：
```swift
var objectWillChange: ObservableObjectPublisher
```
2. 在View中，把其持有的ViewModel对象包装为`@ObservedObject`
```swift
struct ViewStruct: View {
    @ObservedObject var viewModel: ViewModelClass
    ...
}
```
这个包装的意思是，这个对象是一个ObservableObject，每次当它有所改变，View都要马上重新绘制相应的部分。

3. 在ViewModel中，把其持有的Model对象包装为`@Published`
```swift
class ViewModelClass: ObservableObject {
    @Published private var model: ModelStruct = ...
    ...
}
```
这个包装的意思是，每次这个model发生改变，就会自动的调用`objectWillChange.send()`方法，发布一个消息表示有变化发生。
这样一来，当Model变化时，View就能通过ViewModel来得知这个变化，作出响应了。

——从这两个过程可以看出，View和Model之间没有任何直接的连接，仅仅通过ViewModel这个中间人来完成沟通。因此它们都可以把彼此看作一个简单的接口。不需要了解其具体的类型和实现。相比MVC，MVVM达到了“**低耦合**”的要求。大家都知道，降低了代码的耦合，好处会非常的多：易于测试、代码可复用、开发任务上可以更细分...
## 二、类型
### 1. Struct/Class
在上面MVVM的代码中，可以看到ViewModel是一个`class`,而View是一个`struct`。学习Java的同学可能没有接触过Struct，它们两个是很类似的，但又有一些区别。
#### 共同点
- 都具有一些属性，包括`var`(变量)和`let`(常量，不可变的`var`)
- 都具有一些`func`(函数)（即Java中的“方法”）
- 都有initializier，写作`init`，类似构造器

注意：Swift中函数参数的语法和Java中也有所区别：
```swift
func multiply(operand: Int, by: Int) -> Int {
  return operand * by
}
multiply(operand: 5, by: 6)  
// 如果参数有label，调用时要标注label

func multiply(_ operand: Int, by otherOperand: Int) -> Int {  
  return operand * otherOperand
}
multiply(5, by: 6)
// 参数可以有两个label
// 前一个（by）调用者使用
// 后一个（otherOperand）函数内使用
// 下划线_代表没有label
```
#### 不同点
|Struct|Class|
|--|--|
| copy on write | 自动引用计数 |
| functional programming | OOP |
| 没有继承 | 单继承 |
| 自带`init`初始化所有变量 | `init`不自动初始化变量 |
| 可变性（mutability）必须标明 | 总是可变的 |
需要注意的：
   - 方法参数和赋值传递的如果是值类型，则只是传递了一个copy，而不是对象本身的引用。
   - 对于struct，如果一个函数中需要修改`self`（类似Java中的`this`）的属性，则该函数必须被标注为`mutating func`
   - 大部分东西都是struct，但是MVVM中的ViewModel是一个class

对于初学者，其中一些概念可能不能马上明白，但是在后面的学习中，会渐渐理解这些区别具体体现在哪里。
### 泛型
Swift也是强类型语言，所以这点和Java一样，用泛型来表示“不在意”的类型。不必多说，最简单的例子就是数组：
```Swift
struct Array<Element> {
    ...
    func append(_ element: Element) { ... } 
}
var a = Array<Int>()
```
### 函数类型化
函数类型化，即可以将一个变量设置为“函数”类型，也就是可以将一个函数赋给一个变量。如果了解过Java 8的Lambda表达式，应该比较好理解这个特性。


```Swift
 //  以下例子都可以作为函数变量的类型
(Int,Int)->Bool //得到两个Int，返回一个Bool
(Double) -> Void // 得到一个Double，无返回值
() -> Array<String> // 无参数，返回一个字符串数组
() -> Void // 无参数无返回值

 //  实际声明一个变量时：
var foo: (Double) -> Void // 变量foo的类型是：一个参数为一个Double，无返回值的函数
func doSomething(what: () -> Bool) // 变量what：一个无参数，返回Bool的函数
```
实际使用中：
```Swift
func square(operand: Double) -> Double {
      return operand * operand
}
var operation: (Double) -> Double
operation = square // 将函数square赋给变量operation
let result1 = operation(4) // result1 = 16
operation = sqrt // sqrt是内部的一个开方函数，也是获取并且返回一个Double
let result2 = operation(4) // result2 = 2
```
既然函数可以作为一个变量，那么它也可以作为另一个函数的参数。这种操作可以把代码简化的非常少，我们一步一步看：
- 最清晰但也最复杂的方法：先定义函数，再把它作为参数传入另一函数：
 ```Swift
  func doubleInt(num: Int) -> Int {
    return num * 2
}
func calculator(operand: Int, operation: (Int) -> Int) -> Int {
    return operation(operand)
} //记住这个函数，它的第二个参数是一个参数为一个int，返回值为int的函数（好绕口）
let ans = calculator(operand: 3, operation: doubleInt) //调用时传入函数doubleInt
  ```
- 但我们可以把它简化为，作为参数传入时再定义函数：

```Swift
let ans = calculator(operand: 3, operation: {(num: Int) -> Int in
    return num * 2
}) // doubleInt函数直接在参数列表里定义了，连名字都不用起
// 注意要用大括号包裹
  ```

- 进一步省略类型声明和括号：

```swift
let ans = calculator(operand: 3, operation: {num in
    return num * 2
})
// 为什么可以省略num的类型和返回值的类型？因为我们在定义calculator方法时已经在参数列表里定义过了！
```

 - 省略return，因为operation是calculator的最后一个参数，可以用大括号挂到小括号外：

```Swift
let ans = calculator(operand: 3) {num in num * 2}
  ```
如果没有了解过这个过程，乍一看这一行代码，实在是令人疑惑啊...
## 三、协议
Swift中的协议（protocol），类似于java中的接口（interface）。和interface一样，它可以代表一个类型，并且定义了这个类型该具有什么样的能力（行为），同时其他人也可以用这种方式来表示他需要的是什么样的能力（行为）。但是具有这样能力的class或struct究竟是什么，他们具体是怎么实现这种能力的，都不会暴露出来。

```swift
protocol Moveable {
      func move(by: Int)
      var hasMoved: Bool { get }
      var distanceFromStart: Int { get set }
}
```

一个协议里面定义了变量和函数，但是没有实际的初始化和实现，可以被别的struct/class来实现。

```swift
struct PortableThing: Moveable {
    //这里必须实现move(by:)、hasMoved和distanceFromStart
}
```

这也是“functional programming”的思想，关注“功能”，而隐藏如何实现的细节。这类似OOP中的“封装”。
### 实现和扩展

- 实现

首先，协议也可以实现协议。相当于继承，但不需要提供实现，但可以增加新的成员。

```swift
protocol Vehicle: Moveable {
      var passengerCount: Int { get set }
  }
class Car: Vehicle {
    // 这里必须实现move(by:), hasMoved, distanceFromStart以及passengerCount
 }
```

类似java，一个类可以实现多个协议。

```swift
class Car: Vehicle, Impoundable, Leasable {
    //必须实现每个协议中定义的每个变量和函数
}
```

实现之后，这个协议也可以作为一个类型。声明为这个类型的对象可以调用协议中的函数。

> "Constrain and gain"
> 实现一个协议，则被**约束**一定要实现它的所有函数，但也**拥有**了它的所有能力。

- 扩展

对协议的扩展是给协议中的某个成员提供默认的实现。后续再来实现这个协议的类可以选择不实现那些成员而是使用扩展中的默认实现，也可以自己重新实现（类似override）。

```swift
protocol Moveable {
      func move(by: Int)
      var hasMoved: Bool { get }
      var distanceFromStart: Int { get set }
  }
extension Moveable {
      var hasMoved: Bool { return distanceFromStart > 0 }
  }
struct ChessPiece: Moveable {
    // 这里只需要实现move(by:)和distanceFromStart
    // 不需要实现hasMoved，因为扩展中定义了一个默认的实现
    // 但是如果想自己实现也行
}
```

同时，extension也可以实现别的协议，例如：

```Swift
struct Boat {
  ...
}
extension Boat {
    func sailAroundTheWorld() { /* implementation */ }
}
extension Boat: Moveable {
    // 这里要实现move(by:)和distanceFromStart
}
//在扩展boat的同时，也实现了Moveable，所以要在这个对boat的extension里实现Movable的方法
```

### 泛型+协议
和泛型一同使用会使协议的功能更加强大。

```swift
protocol Greatness {
    func isGreaterThan(other: Self) -> Bool 
}
```

首先我们写一个这样的协议，表示一种可以比较大小的能力。

注意这个`Self`，代表实际上实现了这个协议的那个类型。

实现了这个协议的类型的对象可以被放进类型为`Greatness`的数组中，此时我们并不需要知道他们究竟是什么class/struct，只需要知道他们拥有`isGreaterThan`方法。

因此，可以写这样一个针对【元素类型为`Greatness`的数组】的extension，来给它定义一个找到最大值的方法。

```swift
extension Array where Element: Greatness {
    var greatest: Element {
    // 在这里for循环遍历所有元素
    // 在这里我们知道所有元素都一定是都实现了Greatness协议的～
    // 所以我们就可以调用他们的isgreaterThan函数，找到最大的那个！
    // return 最大的那个数  
  }
}
```

## 四、布局
### View
在SwiftUI中，几乎所有东西都是View。它们是可以显示在屏幕上面的组件。

```swift
struct ContentView: View {  //这个struct具有view的行为
    var body: some View {
        Text("Hello, world!") 
            .padding()
    }
}
```
这里这个叫做ContentView的struct，以及Text，都是View。
- `some View`：可以有任何type的成员，只要它有view的行为，例如上面这个例子里的Text。
### Layout 布局
View是如何在屏幕上分配空间的呢？
首先View在使用中的结构是这样的，熟悉组件模式的同学应该很容易理解，一个父View中可以包含其他的子View，子View也可以作为父View包含其他的View...
相应的，分配空间的过程也和这样的父子关系相关，概括地说是这样的：
1. 父view（容器）会为其内部的子view提供一定的空间
2. 子view可以参考这个空间，设定他们自己想要的大小，返回给容器
3. 然后容器会根据这个大小将子view放置在自己内部。
### Stack（HStack/VStack/ZStack）
Stack是非常常用的容器，包括`Vstack`（vertical垂直布局）, `HStack`（Horizontal水平布局），`ZStack`（子View上下堆叠）,会将他们的空间以某种规则分割，再提供给各个子view。参考[SwiftUI - VStack, HStack 和 ZStack差异](https://www.jianshu.com/p/d75da37296f0)。
- Stack会根据子view是否“灵活”，来决定为它们提供空间的顺序：
  - 首先提供给最不灵活的子视图：例如：图片`Image`，希望具有固定大小。
  - 稍微灵活一点的例如：文本`Text`，希望大小可以适合文本内容。
  - 最灵活的例如：`RoundedRectangle`(圆角矩形)，可以使用提供给他的任何大小的空间。

  但是，也可以通过修改<u>优先级</u>来决定分配空间的方式：`layoutPriority`
  ```swift
  HStack {
      Text(“Important”).layoutPriority(100) // 参数可以是任何浮点数
      Image(systemName: “arrow.up”) 
      Text(“Unimportant”) //如果Text没有得到足够空间，会被缩略成”...“
  }  // 默认的优先级是0
  ```
一个子view获取到空间后，其大小会从父view可用空间中移除，然后父view会继续考虑下一个子view。
- `Spacer(minLength: CGFloat)`，不会绘制任何东西，作为占位留白。
- `Divider()`，画一条分割线，方向和Stack一致。
- **对齐**：通过Stack的参数来指定
  - `VStack(alignment: .leading) {...}`左对齐
  - 此外还有`.center`, `.top`, `.training`等等

- 一些函数，例如`.padding()`等等，会对View的各个方面起到编辑的作用，可以称为“**Modifier**”（编辑器），有些也会影响布局。
  注意：这些函数都会返回一个`View`，即经过它们修改后的`View`，或者包含原view的容器。
  - 例：`aspectRatio`, 调整视图宽高比。参数1:宽高比，参数2: .fill/.fit
    ```swift
    func aspectRatio(_ aspectRatio: CGFloat? = nil, contentMode: ContentMode) -> some View
    ```
### GeometryReader
可以将`GeometryReader`包裹在实际想显示的视图外：
```swift
var body: View {
     GeometryReader { geometry in 
     ...
    }
}
```
这个`geometry`本质是一个`GeometryProxy` ：
```swift
struct GeometryProxy {
    var size: CGSize
        func frame(in: CoordinateSpace) -> CGRect
    var safeAreaInsets: EdgeInsets
}
```
-  `geometry.size`父容器建议的尺寸（CG：core graphics）
-  `geometry.frame(in: CoordinateSpace)`坐标位置，CoordinateSpace指坐标空间，例如`global`。可以获取它的相对位置。
- GeometryReader总是使用它**被给予的最大的空间**。
> 学习视频：[【SwiftUI】搞懂GeometryReader](https://www.zhihu.com/zvideo/1217544012397305856) 
>
> 这个视频示范了使用GeometryReader，在一个ScrollView中获得当前条目的相对位置，转化成旋转角度，形成一个列表随着滑动操作可以3D旋转的效果，非常有趣。
### Safe Area安全区 
- 一般来说，当分配给一个view空间时，不会包括“安全区”。
  - 最明显的不安全区：IphoneX和之后的机型的“刘海”...
- 边缘等周边的view可能会在安全区内，但是内部的view不应该在安全区内。
  - 但是你也可以选择忽略这一点：
    对Stack调用`.edgesIgnoringSafeArea（[.top]）`表示在上方边缘的安全区绘制内容
### Container容器
容器是如何为子view提供空间的呢？
- `.frame(...)`函数决定空间大小（参数很多，见文档）
- `.position(CGPoint)`决定位置，Stack会根据对齐方式、spacing等等，为每个子View决定这个CGPoint
- `.offset(CGSize)`可以进行偏移

通过这些modifier，我们也可以定义我们自己的想要的容器的效果。

## 五、计算属性

上次学过基本的概念后，我们知道在Swift的Struct和Class中可以定义一些属性，而在Swift中，这些属性还分为**存储属性**和**计算属性**两种。

### 存储属性（stored property）

存储属性就是大家最熟悉的，在一个class或struct的实例中存储的可以直接赋值的变量或常量。变量用`var`定义，常量用`let`定义。

### 计算属性（computed property）

虽然我们也可以把计算属性看成一个逻辑上的**属性**，但是**它不会存储任何值**，而是提供一个`getter`    ，和一个`setter`（非必须），来间接的获取和设置其他属性或者变量的值。

它存储的数据一定是和其他属性存在一定**联系**的，也就是说通过其他属性一定能计算出它。当我们想get它的值时，它会临时对其他存储属性进行计算获得自己的值返回出来，set它的值时，也仅仅是通过改变了其他的存储属性。

我的理解是：它总是通过操作别的属性来“假装“自己也是个存了数据的变量。

官方文档中的例子很便于理解：

```swift
struct Point {
    var x = 0.0, y = 0.0
}
struct Size {
    var width = 0.0, height = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set(newCenter) {
            origin.x = newCenter.x - (size.width / 2)
            origin.y = newCenter.y - (size.height / 2)
        }
    }
}
var square = Rect(origin: Point(x: 0.0, y: 0.0),
                  size: Size(width: 10.0, height: 10.0))
let initialSquareCenter = square.center
square.center = Point(x: 15.0, y: 15.0)
print("square.origin is now at (\(square.origin.x), \(square.origin.y))")
// "square.origin is now at (10.0, 10.0)"
```

> 以下整理自[官方文档](https://docs.swift.org/swift-book/LanguageGuide/Properties.html)：
>
> 这个例子中定义了三种用于处理几何图形的结构：
>
> - `Point`点，封装一个点的x坐标和y坐标
> - `size`大小，封装宽度和高度。
> - `Rect`矩形，通过一个原点`origin`（矩形左下角的点）和大小`size`来定义一个矩形，这两个属性的类型分别就是上面定义的`Point`和`Size`。
>
> ---
>
> `Rect`结构中还有一个**计算属性**`center`，通过它我们可以：
>
> - `get`：利用一个矩形的另两个属性计算出它的中心点位置，返回代表中心点的`Point`实例
> - `set`：获取一个`Pointer`实例作为新的中心点，修改另外两个属性来将矩形移动到新的位置上。

上面代码的最后，我们先设定了一个原点为point(0,0)，长宽为size(10,10)的矩形，这时获取`square.center`，我们会获取到(5,5)。这个值其实并没有被储存在实例`square`中，而是在我们获取的时候，自动调用了`get`把答案计算出来了！然后，我们把`square.center`设置成了point(15,15)，相当于移动了这个矩形的中点，这个值也没有被储存，而是自动调用了`set`，计算出了移动后的`origin`，

总的来说，虽然我们在`Rect`的实例中定义了类型为`Point`的中心点`center`，我们也能设定和获取这个中心点的值，但是实际上`Rect`的实例是**不储存任何关于中心点的数据**的，但我们可以随时“**计算**”出这个数据（<u>通过对size和origin进行运算</u>），也可以直接**设定**这个数据（<u>实际也只是修改了origin</u>）。

## 六、闭包Closure

### 概念

首先，“闭包”的概念已经有很多人做过解读。简单的说就是：闭包是将**函数** 与**环境** 打包存储在了一起。

**环境**是函数的上下文（context），可以理解成在函数的外部作用域定义的那些变量（或常量）。以往我们理解的函数是不能使用自己作用域之外定义的变量的。但是对于闭包则不是这样，闭包把它创建时的**外部环境**给打包起来了。

例如我们在闭包函数之外的作用域定义过一个变量a，在没有将a作为参数传入闭包的情况下，闭包作用域内依然可以使用这个变量：

```swift
var a = 1
func add () -> Int {
    return a+1
}
```

我们把这种行为称为“**捕获**”，也就是说我们的闭包获取了外部的变量和常量，它已经悄悄记录了`a = 1`这件事。

### Swift中的闭包

那么对于Swift，我们首先要知道的是，**在Swift中函数都是闭包，或者可以说它的函数就是闭包的一种。** 也就是说，所有的函数都具有这种捕获的能力。

在上篇笔记[Swift学习笔记1](http://km.oa.com/articles/show/478636)里有提到，在Swift中，函数可以作为一种类型，可以被赋给变量或者作为参数传递给别的函数。例如上面的例子中的`add`函数，当我们把它作为参数传递到别的函数中去执行，理论上它已经完全脱离了创建时的环境，但是真正调用运行时，它依然知道`a`的值。

#### 语法

上篇笔记中也展示了在Swift中如何将”把函数类型的变量作为参数传递“的代码不断简写的过程，其实，这个过程就时在把一个标准的函数语法，转变成了简单的闭包语法。我们可以比较一下这两种语法：

```swift
// 函数
func 函数名(参数1, 参数2 ...) -> 返回类型 {
    代码块
}

// 闭包
{ (参数1, 参数2 ...) -> 返回类型 in    
    代码块
}
```

观察两者的差别，标准的闭包写法就是在函数中删掉了`func`和函数名，把参数列表和返回类型挪到了大括号中，再在函数体前加了一个`in`。

了解之后，我们再来看上篇笔记中的例子：

```swift
func calculator(operand: Int, operation: (Int) -> Int) -> Int {
   return operation(operand)
}
// 简化前：定义标准的函数传入
func doubleInt(num: Int) -> Int {
   return num * 2
}
let ans = calculator(operand: 3, operation: doubleInt) 

// 简化后：当场写一个闭包传入
let ans = calculator(operand: 3) {num in num * 2}    
```

最后一行中的大括号`{num in num * 2}`其实就是闭包写法的简化版`doubleInt`函数。

在SwiftUI中，我们会大量的使用闭包，使得代码看起来非常简洁。例如：

```swift
struct Text: View {
    var body: some View {
        Button(action: { 
          print("Button Click" ) // 这就是一个闭包中的代码
        }) {
            Text("Hello World!")
        }
    }
}
```

### 闭包逃逸

当我们把一个闭包作为参数传入到了一个函数中，可能会出现这种情况：**在函数执行结束前，这个闭包并没有被执行**，而是仅仅被赋给了一个变量或属性，被“存”起来了。也就是说这个闭包又被传递到了函数之外，这种情况就称为闭包从函数中“逃逸”了。

上篇中提到的网课Demo中有这样的一块代码：

```swift
struct Grid<Item, ItemView>: View where Item: Identifiable, ItemView: View{
    var items: [Item]
    var viewForItem: (Item) -> ItemView
    
    init(_ items: [Item], viewForItem: @escaping (Item) -> ItemView) { // 注意这个escaping
        self.items = items
        self.viewForItem = viewForItem
    }
  ... ...
}
```

可以看到在`init`函数中，传入的函数`viewForIntem`没有被马上执行，而是赋给了属性`self.viewForItem`。因此它是一个逃逸的闭包，这时，我们要在参数列表上为它打上一个`@escaping`标记。在Swift3之后的版本中，闭包参数默认都是非逃逸的，要对逃逸的闭包额外标记。

## 七、Enum枚举

枚举的概念大家都非常熟悉，简单的说就是定义一种包含有限种值的类型。比方说你在一家餐厅看菜单点菜，菜单上只有四种菜，所以你点的只能是这四种之一，如果你点菜单上没有的菜，饭店就会觉得你有问题。

```swift
enum FastFoodMenuItem {
  case hamburger
    case fries
    case drink
    case cookie 
}
```

在Swift中，Enum和Struct一样也是值类型（value type），所以它在传递的时候也是用拷贝的方式。

### 关联数据

枚举中的每一个值（我们可以把一种值称为一种状态（state））都可以有一些和它相关联的数据，类似于一个状态具有的特殊属性：

```swift
enum FastFoodMenuItem {
    case hamburger(numberOfPatties: Int)
    case fries
    case drink(brand: String, ounces: Int) 
    case cookie 
}
```

在使用的时候，如果某个状态有这种相关数据，我们必须在声明这个变量时，也给到这些数据的值：

```swift
let menuItem: FastFoodMenuItem = FastFoodMenuItem.hamburger(patties: 2)
var otherItem: FastFoodMenuItem = FastFoodMenuItem.cookie
```

声明时这两种语法都可以使用：

```swift
let menuItem = FastFoodMenuItem.hamburger(patties: 2) 
var otherItem: FastFoodMenuItem = .cookie

var yetAnotherItem = .cookie // 这样不行！总得提到FastFoodMenuItem，不然Swift不知道
```

检查某个枚举类型的变量的具体状态的时候，可以用`switch`，每个状态对应一个case，这个就不用多说了。不过，我们的每个状态如果都有相关数据的话，我们在响应的case中是可以获取到这些数据的：

```swift
var menuItem = FastFoodMenuItem.drink(“Coke”, ounces: 32) 
switch menuItem {
    case .hamburger(let pattyCount): print(“a burger with \(pattyCount) patties!”) 
  case .fries(let size): print(“a order of fries!”)
    case .drink(let brand, let ounces): print(“a \(ounces)oz \(brand)”)
    case .cookie: print(“a cookie!”)
 } //a 32oz Coke!
```

### 方法和协议

另外，Enum本身没有任何存储的属性，但是里面可以有**方法**。比方说这样一个方法：

```swift
enum FastFoodMenuItem { ...
    func isIncludedInSpecialOrder(number: Int) -> Bool {
        switch self {
            case .hamburger(let pattyCount): return pattyCount == number
            case .fries, .cookie: return true 
           case .drink(_, let ounces): return ounces == 16 
         }
    } 
}
```

在方法里Enum可以`switch`自己（`self`）。这个`self`类似类中表示当前实例的`self`，表示调用方法的当前状态变量。

此外，**Enum是可以实现协议的**。和Struct实现协议的方式一样，在开头声明实现的协议名，然后它就能获得这个协议定义的能力，同时也需要实现其中的方法。例如，我们写一个实现了`CaseIterable`协议的Enum：

```swift
enum example: CaseIterable {
     // case ...
}
```

`CaseIterable`可以使这个Enum自动拥有了一个静态变量`allCases`。它是一个包含这个Enum中所有Case的Iterator。感觉也会非常好用。

## 八、Optional

`Optional`是Swift中的一种特殊类型。用来处理一个值可能存在缺失、不确定的情况。在了解了枚举之后，我们可以将`Optional`基本理解为这样的一个带泛型的枚举：

```swift
enum Optional<T> { 
    case none
    case some(T) 
}
```

它的两个值分别是“有”和“没有”。对于“有”的状态我们可以给它关联上某个`T`类型的数据。这样一来，当我们需要一个有时候会“不确定”的值，或者是想把一个值定义为“未决定”，就可以用这个`Optional`类型，当这个值尚未决定或者没有准确答案的时候，就让它为`none`，表示“没有值”，如果决定了，就为附带上实际结果`T`的`some`。

### 声明Optional

在类型后加一个小问号`？`就可以声明一个`Optional`类型了。这个问号就可以理解为一种不确定：这里存着一个字符串吗？有可能，但是不一定哦。这下面三组声明方式，每组内两个的作用都是一样的。不赋值、赋`nil`都代表`none` case，赋值则代表`some` case。

```swift
var hello: String?
var hello: Optional<String> = .none // 不初始化的话，默认为不存在

var hello: String? = “hello” 
var hello: Optional<String> = .some(“hello”) 

var hello: String? = nil
var hello: Optional<String> = .none
```

### 使用Optional

要注意这样的方式声明的是一个`Optional`，而不是一个`String`！那么如果我们想把字符串取出来用的时候要怎么办呢？

- 强制解析：

  在Option的变量名后面加一个感叹号`!`可以强行获取其中的值。但是，这只适用于这个值一定存在，也就是是`some`的情况。如果它不存在，也就是`none`，就会抛出异常使你的程序崩溃。

  ```swift
  let hello: String? = ...
  print(hello!)
  ```

  **按规范来说，强制解析一般是不可以使用的。** 但是它也有存在的意义，如果说在某个场景中，这个值本就不应该不存在，那么如果我们强制解析它，当值不存在时程序就可以及时的崩溃的，而不会使程序自己默默的承受这个错误而产生更大的问题。**不过目前在正式的项目中不要这么用。** 

- 可选绑定（安全）

  这种方式是判断Option是否包含值。如果包含，就把它赋给一个临时的常量或变量。

  ```swift
  if let safeHello = hello {
    print(safeHello)    // 值已经在safeHello中了，可以拿来用
  } else {
    // 没有值的话会进这里 
  }
  ```

其实，`Optional`在Swift中的使用非常广泛。例如，在数组`Array`中，有一个自带的属性叫做`first`。它会返回一个`Optional`实例。如果这个数组非空，则这个实例会包含这数组中的第一个元素。如果数组为空，则这个`Optional`中没有值，为`nil`。我们自己写代码遇到有类似需求的地方也可以考虑使用这种方式来返回结果。

### ??

Optional还可以配合`??`这个符号使用：

```swift
let x: String? = ...
let y = x ?? "foo"
```

这第二句的意思是，如果Option`x`有值，则将它的值赋给`y`。否则，将`“foo”`赋给`y`。

## 九、Equatable协议

大家应该都很熟悉两个等号连用`==`，在很多语言中他都用来比较两边的两者是否完全相等。在Java中，它可以用来比较两个基本类型变量的值是否相等，或是引用类型变量的引用地址是否相等。

在Swift中，它的功能也是如此，但是不同的是，Swift中的`==`其实是一个**函数**，它的参数是两个元素，返回值是一个布尔值。这个函数是由一个重要的协议`Equatable`提供的。（上一篇笔记中有讲过协议的概念，就是类似于Java中的接口。）这个协议中主要的方法有两个：

```swift
static func == (Self, Self) -> Bool
static func != (Self, Self) -> Bool
```

其中只有第一个方法是使用者必须要去实现的。它类似于我们在Java中重写`equals()`，你可以让你的类型实现这个协议，并且自己定义一个标准来判断在逻辑上这两个实例是否相等。

所以如果要使用`==`，元素的类型必须是一个`Equatable`的类型。如果你对一个泛型的属性去用`==`是会报错的，因为这个泛型是否`Equatable`，Swift并不知道。必须要声明泛型的类型一定是`Equatable`才可以，例如这样：

```swift
struct MemoryGame<CardContent> where CardContent : Equatable {
  ... ...
}
```

当你在使用泛型时，想在一定程度上限制泛型的类型，就可以用这种语法。它类似于Java中用通配符表示的有界的泛型`<? extends E>`。

## 十、Group

在SwiftUI中有一个容器叫做`Group`，它不是一个会显示出来的组件，而是一个给子view分组的工具。Swift限制一个容器中，最多只能有10个子view，不然就会报错。同时，子view太多也可能会难以管理。因此，我们把子view分别装到不同的Group中，就可以在完全不影响UI的情况下将它们进行分组。

```swift
var body: some View {
    VStack {
        Group {
            Text("Hi")
            Text("Hi")
            Text("Hi")
            Text("Hi")
            Text("Hi")
            Text("Hi")
        }
        Group {
            Text("Hi")
            Text("Hi")
            Text("Hi")
            Text("Hi")
            Text("Hi")
        }
    }
}
```


