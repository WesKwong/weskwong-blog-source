---
author: ["WesKwong"]
date: 2024-08-03T11:20:37+08:00
title: "SwiftUI自定义状态管理：从头开始构建@State"
description: "通过利用SwiftUI提供的自定义状态管理API，构建一个简易的`@State`，拥有与官方实现相近的行为，从而初步探究SwiftUI的自定义状态管理能力和`@State`的实现原理。"
summary: "通过利用SwiftUI提供的自定义状态管理API，构建一个简易的`@State`，拥有与官方实现相近的行为，从而初步探究SwiftUI的自定义状态管理能力和`@State`的实现原理。"
tags: ["SwiftUI"]
categories: ["Coding"]
# cover:
#   image: images/cover.png
showtoc: true
---
在SwiftUI状态管理中，`@State`作为数据源的定义，极为重要，然而SwiftUI作为一个闭源框架，仅对外公开了`@State`的定义：

```swift
@frozen @propertyWrapper public struct State<Value> : DynamicProperty {

    /// Initialize with the provided initial value.
    public init(wrappedValue value: Value)

    /// Initialize with the provided initial value.
    public init(initialValue value: Value)

    /// The current state value.
    public var wrappedValue: Value { get nonmutating set }

    /// Produces the binding referencing this state value
    public var projectedValue: Binding<Value> { get }
}
```

而没有公开内部的实现，因此本文将通过利用SwiftUI提供的自定义状态管理API，构建一个简易的`@State`，拥有与官方实现相近的行为，从而初步探究SwiftUI的自定义状态管理能力和`@State`的实现原理。

## 属性包装器Property Wrapper

### 使用背景

为了更好地理解Property Wrapper，假设我们需要向属性添加日志记录的功能：当我们**写入**一个属性时，打印原有的旧值和即将写入的新值：

```swift
struct Storage {
    private var _x: Int = 0

    var x: Int {
        get { return _x }
        set {
            print("Set value: \(_x) -> \(newValue)")
            _x = newValue
        }
    }
}

var storage = Storage()

storage.x = 1
storage.x = 10
storage.x = 100

/* the app will print:
 Set value: 0 -> 1
 Set value: 1 -> 10
 Set value: 10 -> 100
*/
```

当`storage`中拥有更多的属性时，想要向这些属性添加日志记录功能将会使代码变得十分冗长：

```swift
struct Storage {
    private var _x: Int = 0
    private var _y: Int = 0

    var x: Int {
        get { return _x }
        set {
            print("Set value: \(_x) -> \(newValue)")
            _x = newValue
        }
    }

    var y: Int {
        get { return _y }
        set {
            print("Set value: \(_y) -> \(newValue)")
            _y = newValue
        }
    }
}

var storage = Storage()

storage.x = 1
storage.x = 10
storage.x = 100
storage.y = 1
storage.y = 10
storage.y = 100

/* the app will print:
 Set value: 0 -> 1
 Set value: 1 -> 10
 Set value: 10 -> 100
 Set value: 0 -> 1
 Set value: 1 -> 10
 Set value: 10 -> 100
*/
```

我们可以考虑将这个功能包装成一个新类型，该新类型拥有日志记录功能：

```swift
struct logger<T> {
    private var _val: T

    init(val: T) { _val = val }

    var val: T {
        get { return _val }
        set {
            print("Set value: \(_val) -> \(newValue)")
            _val = newValue
        }
    }
}

struct Storage {
    private var _x: logger<Int> = logger<Int>(val: 0)
    private var _y: logger<Int> = logger<Int>(val: 0)

    var x: Int {
        get { return _x.val }
        set { _x.val = newValue }
    }

    var y: Int {
        get { return _y.val }
        set { _y.val = newValue }
    }
}

/* The rest remains unchanged... */
```

### 使用方法

然而，这样的写法依然非常冗长，Swift 5提供了一个新特性Property Wrapper，他可以使得自定义的新类型成为一个自定义的新属性包装器：

```swift
@propertyWrapper
struct logger<T> {
    private var _val: T

    init(wrappedValue: T) {
        _val = wrappedValue
    }

    var wrappedValue: T {
        get { return _val }
        set {
            print("Set value: \(_val) -> \(newValue)")
            _val = newValue
        }
    }
}
```

由`@propertyWrapper`包装的新类型必须具有`wrappedValue`属性，被这个新包装器包装的属性，将会被存储到包装器内部的`_val`中，而向外表现出来的行为，例如`get`和`set`，将会被`wrappedValue`所代理：

```swift
struct Storage {
  	// @propertyWrapper 具有两种初始化的方法：
    @logger var x = 0 // 由编译器隐式调用`init(wrappedValue: T)`
    @logger(wrappedValue: 0) var y // 显式指定wrappedValue初始化值
}
```

可以看到，`@propertyWrapper`帮助我们将冗长的逻辑缩短为仅需额外使用一个包装器符号即可。

### 访问包装器实例

除此之外，包装器内还可以定义一些额外的工具函数：

```swift
@propertyWrapper
struct logger<T> {
    private var _val: T

    init(wrappedValue: T) { /* ... */ }

    var wrappedValue: T { /* ... */ }

    func greet() { print("I'm a logger!") }
}
```

在变量名前添加下划线`_`为包装器实例，通过访问包装器实例，我们可以调用包装器中定义的工具函数：

```swift
storage._x.greet()
```

但是直接从外部调用会产生编译错误：

`error: '_x' is inaccessible due to 'private' protection level`

这是因为包装器实例默认为`private`属性，我们可以通过`projectedValue`来访问公开更多对外的API，从而实现访问包装器实例：

```swift
@propertyWrapper
struct logger<T> {
    private var _val: T

    init(wrappedValue: T) { /* ... */ }

    var wrappedValue: T { /* ... */ }

    var projectedValue: logger<T> { return self }

  	func greet() { /* ... */ }
}
```

此时我们可以通过Swift提供的语法糖访问`projectedValue`：

```swift
storage.$x.greet()
// I'm a logger!
```

经过上面的学习，我们可以开始尝试动手构建`@State`了，参考官方定义，我们可以尝试简单地定义包装器`@MyState`：

```swift
import SwiftUI

@propertyWrapper
struct MyState<T> {
    private var _val: T

    init(wrappedValue: T) {
        _val = wrappedValue
    }

    var wrappedValue: T {
        get { return _val }
        set { _val = newValue }
    }
}
```

然后使用我们定义好的`@MyState`：

```swift
struct MyView: View {
    @MyState private var msgFlag: Bool = true

    var body: some View {
        Text(msgFlag ? "Hello, World!" : "Hello, Swift!")

        Button("Click Me") {
            if msgFlag { msgFlag = false }
            else { msgFlag = true }
        }
    }
}
```

然而，此时会产生编译报错：

`error: cannot assign to property: 'self' is immutable`

这个报错的原因是结构体`MyView`是不可变的，属性`msgFlag`（`MyState`包装器实例）属于`MyView`，然而我们的操作`msgFlag = false`会改变`msgFlag`，即改变了`MyState`实例，从而改变了`MyView`。

## 不可变标记nonmutating

参考官方定义`public var wrappedValue: Value { get nonmutating set }`，我们发现需要在`wrappedValue`中加入**不可变标记`nonmutating`**，使得`MyState`实例向`MyView`承诺，向自己的赋值操作不会改变自己：

```swift
@propertyWrapper
struct MyState<T> {
    ...
    var wrappedValue: T {
        ...
        nonmutating set { _val = newValue }
    }
}
```

此时，编译错误的定位位置从`MyView`的`msgFlag = false`处移动到了`MyState`的`_val = newValue`处，因为虽然`MyState`对外承诺不会改变自己，但是实际上却对自身内部属性`_val`执行了赋值操作。

虽然我们此时仍未能编译成功，但是我们将问题变成了：

**如何在赋值时不改变`_val`属性？**

## 引用类型Reference Type

为了解决这个问题，我们很容易联想到C语言中的**指针类型**，即结构体内只保存指向属性实际存储地址的指针，在赋值的时候，我们改变的是实际存储地址中的值，而结构体本身的指针并不会改变。

在Swift中，数据类型分为两种：

- **值类型(Value Types)**：每个实例保留一份独有的数据拷贝，一般以结构体`struct`、枚举`enum` 或者元组`tuple`的形式出现。
- **引用类型(Reference Type)**：每个实例共享同一份数据来源，一般以类`class`的形式出现。

两种类型的区别如下：

```swift
// 值类型
struct val_type_data<T> {
      var val: T
      init(_ initval: T) { val = initval }
}
var x = val_type_data(1)
var y = x
y.val = 2
print("x: \(x.val), y: \(y.val)")
// x: 1, y: 2
```

```swift
// 引用类型
class ref_type_data<T> {
  	var val: T
  	init(_ initval: T) { val = initval }
}
var x = ref_type_data(1)
var y = x
y.val = 2
print("x: \(x.val), y: \(y.val)")
// x: 2, y: 2
```

可以看到，在两份代码中，都是变量`y`复制自变量`x`，然后变量`y`改变内部属性`val`的值：

- 在值类型中，变量`y`实际上是独立的一份拷贝，不影响变量`x`的值
- 在引用类型中，变量`y`实际上和变量`x`共用同一份数据实例，因此改变`y`的同时，也改变了`x`

通过上面的学习，我们可以为`MyState`中的`_val`**创建一个存储空间**，使得我们在赋值时，不改变`_val`本身：

```swift
class Storage<T> {
  	var val: T
  	init(_ initval: T) { val = initval }
}
```

然后我们让`MyState`利用这个**存储空间**：

```swift
@propertyWrapper
struct MyState<T> {
    private var storage: Storage<T>

    init(wrappedValue: T) {
        storage = Storage(wrappedValue)
    }

    var wrappedValue: T {
        get { return storage.val }
        nonmutating set { storage.val = newValue }
    }
}
```

**此时，编译成功！**

但是，我们会发现，点击按钮后，**界面的文本并没有任何变化**。此时如果我们将`msgFlag`打印出来，可以发现数据确实已经发生了改变，但是SwiftUI并不知道我们改变了数据，因此没有重新绘制界面，因此，接下来的问题变成了：

**如何通知SwiftUI数据已经改变，以引起界面的重新绘制？**

## 动态属性DynamicProperty

参考官方定义`@frozen @propertyWrapper public struct State<Value> : DynamicProperty`，我们发现为了利用SwiftUI的状态管理能力，我们需要使我们的`MyState`符合`DynamicProperty`协议，使得SwiftUI侦听该包装器：

```swift
@propertyWrapper
struct MyState<T>: DynamicProperty {
    ...
}
```

但此时我们发现界面仍然不会改变，因为我们需要主动通知SwiftUI数据已发生改变，在本文中，我们使用SwiftUI的自定义状态管理能力`ObservableObject`：

```swift
class Storage<T>: ObservableObject {
    var val: T {
        willSet {
            objectWillChange.send()
        }
    }
    init(_ initval: T) { val = initval }
}
```

当我们将要对存储空间中的`val`赋值时，通过`objectWillChange.send()`能力来通知SwiftUI准备重新绘制界面。然后，为了新的`Storage`，我们需要利用SwiftUI的另一个自定义状态管理能力`@StateObject`：

```swift
@propertyWrapper
struct MyState<T>: DynamicProperty {
    @StateObject private var storage: Storage<T>

    init(wrappedValue: T) {
        _storage = StateObject(wrappedValue: Storage(wrappedValue))
    }

  	...
}
```

**此时我们发现，界面成功刷新了！**

## 双向绑定Binding

官方的实现中，除了对当前视图的刷新，还支持和子视图之间的双向绑定，我们直接尝试官方操作：

```swift
struct MyView: View {
    ...
    @MyState private var bindingText: String = "Hello, World!"

    var body: some View {
        ...
        Text("Binding Test Here: " + bindingText)
        TextField("Write something", text: $bindingText)
    }
}
```

显然这会导致编译错误：

`Compiling failed: cannot find '$bindingText' in scope`

参考官方实现`public var projectedValue: Binding<Value> { get }`，并回顾[@propertyWrapper](#属性包装器Property Wrapper)中的内容，会发现这是因为我们并没有实现`MyState`的`projectedValue`属性，因此我们可以进一步利用SwiftUI的自定义状态管理能力，在`MyState`中定义：

```swift
@propertyWrapper
struct MyState<T>: DynamicProperty {
    ...
    var projectedValue: Binding<T> {
        Binding(
          get: { wrappedValue },
          set: { wrappedValue = $0 }
        )
    }
}
```

**此时我们发现，子视图`TextField`可以成功改变父视图`MyView`了！**

至此，我们成功地利用SwiftUI的自定义状态管理能力，构建了一个与官方实现具有相似特性的`@State`包装器，然而，我们可以发现，我们的定义与官方的定义仍有一定的差别，例如缺少了一个`@frozen`包装器，这可能是因为官方实现不仅仅实现了界面的刷新，还考虑了性能问题。

## 附

完整的示例代码如下：

```swift
// 存储数据的存储空间
class Storage<T>: ObservableObject {
    var val: T {
        willSet {
            objectWillChange.send()
        }
    }
    init(_ initval: T) { val = initval }
}

// 自定义的State包装器
@propertyWrapper
struct MyState<T>: DynamicProperty {
    @StateObject private var storage: Storage<T>

    init(wrappedValue: T) {
        _storage = StateObject(wrappedValue: Storage(wrappedValue))
    }

    var wrappedValue: T {
        get { return storage.val }
        nonmutating set { storage.val = newValue }
    }

    var projectedValue: Binding<T> {
        Binding(
          get: { wrappedValue },
          set: { wrappedValue = $0 }
        )
    }
}

// 示例界面
struct MyView: View {
    @MyState private var msgFlag: Bool = true
    @MyState private var bindingText: String = "Hello, World!"

    var body: some View {
        Text(msgFlag ? "Hello, World!" : "Hello, Swift!")

        Button("Click Me") {
            if msgFlag { msgFlag = false }
            else { msgFlag = true }
        }
        Text("Binding Test Here: " + bindingText)
        TextField("Write something", text: $bindingText)
    }
}
```

## Reference

1. [@State 研究](https://fatbobman.com/zh/posts/swiftui-state/)
1. [Swift 5 属性包装器Property Wrappers完整指南](https://juejin.cn/post/6844904018121064456)
1. [Let's build @State](https://www.fivestars.blog/articles/lets-build-state)
1. [Swift里的值类型与引用类型](https://numbbbbb.gitbooks.io/-the-swift-programming-language-/content/chapter4/05_Value_and_Reference_Types.html)