layout: post
tags: 
- iOS
- ReactiveCocoa
date: 2015-09-01 09:32
title: ReactiveCocoa 3.0 初步
---

本文总结了《[A FIRST LOOK AT REACTIVECOCOA 3.0](http://blog.scottlogic.com/2015/04/24/first-look-reactive-cocoa-3.html)》和《[REACTIVECOCOA 3.0 - SIGNAL PRODUCERS AND API CLARITY](http://blog.scottlogic.com/2015/04/28/reactive-cocoa-3-continued.html)》中关于`ReactiveCocoa 3.0`新的Swift API的基本用法。

`ReactiveCocoa 3.0`的Swift API引入了泛型、`pipe-forward`运算符，并且还运用了`curried functions`。新的语言特性让`ReactiveCocoa`变的更好更简洁了。

# Creating Signals
和`Objective-C`版本`ReactiveCocoa`中的`RACSignal`对应的是`Swift`版本中的`Signal`。关于`Signal`很重要的一点是这个类是个泛型类：
```swift
class Signal<T, E: ErrorType> {
    ...
}
```
类型`T`代表的是信号发出的`next` `event`中数据的类型；`E`表示错误的类型必须是符合`ErrorType`协议的。
`Swift`的`Signal`创建的方式和`Objective-C`的`RACSignal`创建方式很相似：
```swift
func createSignal() -> Signal<String, NoError> {
    var count = 0
    return Signal {
        sink in
        NSTimer.schedule(repeatInterval: 1.0) {
            timer in
            sendNext(sink, "tick #\(count++)")
        }
        return nil
    }
}
```
`Signal`的构造函数接受一个`generator`闭包的参数，这个`generator`被调用并且传入一个`sink`，在这个例子里`sink`的类型是`SinkOf<Event<String, NoError>>`。任何传给这个`sink`的`event`都会被`signal`发射出去。
`sendNext`的第二个参数接收传入的值，并且构造成`event`传给`sink`。

# Observing Signals
观察监听一个`Signal`有几种不同的方式，最简单的是用`observe`方法：
```swift
let signal = createSignal()
signal.observe(next: { print($0) })
```
这段代码的输出如下：
```
tick #0
tick #1
tick #2
tick #3
tick #4
```
或者也可以提供一个`Sink`来观察`Signal`的`Event`：
```swift
createSignal().observe(SinkOf {
    event in
    switch event {
    case let .Next(data):
        print(data.unbox)
    default:
        break
    }
})
```
`Event`类型是一个枚举，其中的`Next`和`Error`事件类型包含了其相关值。上面例子中的`SinkOf`构造函数构造了一个类型是`SinkOf<Event<String, NoError>>`的`sink`。
由于`Swift`语言本身的限制，`Event`中封装的`data`是用[`LlamaKit Box Class`](https://github.com/LlamaKit/LlamaKit/blob/e28d7f6e82fbd5dcd5388b36e2acf4eedb44b4e8/LlamaKit/Result.swift#L156)封装起来的，需要用`box/unbox`来处理一下。(注：`Swift 2.0`之后这个限制其实已经没有了)。
# Transforming Signals
`Swift`的`Signal`和`Objective-C`的API另外一个显著的不同点就是对`Signal`的`Transforming`，以`map`操作举例，`map`函数被实现为一个`free function`，而不是`Signal`的成员函数：
```swift
class Signal<T, E: ErrorType> {
    ...
}
func map(signal: Signal, transform: ...) -> Signal
```
然而如果用`free function`做`Signal`的`Transforming`，代码就不再是『流式』的了：
```swift
let transformedSignal = filter(map(signal, { ... }), { ... })
```
所以`ReactiveCocoa`提供了`pipe-forward`运算符（从`F#`借鉴过来的idea）。
`Swift`的`map`实际上是一个`curred function`：
```swift
public func map<T, U, E>(transform: T -> U)
                        (signal: Signal<T, E>) -> Signal<U, E> {
    ...
}
```
也就是说，这个方法第一次调用的时候，我们先提供一个`transform`，从而得到一个新的函数，可以用给定的`transform`把一个`signal`映射为一个新的`signal`。
`pipe-forward`运算符允许我们把`transform`串联起来：
```swift
public func |> <T, E, X>(signal: Signal<T, E>,
                         transform: Signal<T, E> -> X) -> X {
    return transform(signal)
}
```
用`pip-forward`运算符，我们可以把`transforming`写成这样：
```swift
signal
    |> map { $0.uppercaseString }
    |> observe(next: { print($0) })
```

`free functions`比起成员函数有很多好处，其中之一就是不受继承关系的限制。例如`Swift`的`Foundation`定义了`map`函数，可以运用在任何实现了`CollectionType`协议的类或者结构体上，而不需要有任何继承关系。

# Cold and Hot Signals
在之前版本的`ReactiveCocoa`中，`cold signal`和`hot signal`两个概念都是用`RACSignal`来表示的，也经常会造成一些困惑。`ReactiveCocoa 3.0`通过显式的定义两种不同类型（`Signal`和`SignalProducer`）把二者区分开。为了理解`Signal`和`SignalProducer`之间的区别，我们用个实例比较一下。
## Signals
```swift
func createSignal() -> Signal<String, NoError> {
    var count = 0
    return Signal {
        sink in
        print("Creating the timer signal")
        NSTimer.schedule(repeatInterval: 0.1) {
            timer in
            print("Emitting a next event")
            sendNext(sink, "tick #\(count++)")
        }
    }
}

let signal = createSignal()
```
现在我们即使没有为`signal`增加`observer`，它依然在创建和发射`event`：
```
Creating the timer signal
Emitting a next event
Emitting a next event
Emitting a next event
...
```
## Signal Producer
`SignalProducer`的初始化结构和`Signal`很类似：
```swift
func createSignalProducer() -> SignalProducer<String, NoError> {
    var count = 0
    return SignalProducer {
        sink, disposable in
        print("Creating the timer signal producer")
        NSTimer.schedule(repeatInterval: 0.1) {
            timer in
            print("Emitting a next event")
            sendNext(sink, "tick #\(count++)")
        }
    }
}

let signalProducer = createSignalProducer()
```
我们创建了一个`SignalProducer`但是没有任何的`observer`，会发现控制台丽没有任何输出，`timer`也没有被触发。
`SignalProducer`可以看做是个工厂，可以通过调用`SignalProducer`的成员方法`start`，或者通过`pipe-forward`运算符调用`start`这个`free function`创建一个`Signal`：
```swift
signalProducer
    |> start(next: {
        print($0)
    })
```
在上面的例子中，对于`Signal`，即使添加了多个`observer`，`timer`也只会触发一次；对于`SignalProducer`，每次`start`都会创建一个新的`timer`。
通常我们会用`SignalProducer`表示一个操作或者任务，`start`一个`SignalProducer`会启动这个操作；而`Signal`用来表示事件流，无论是否有`Observer`监听。`SignalProducer`非常适合于网络请求，而`Signal`则非常适合表示UI的事件流。

# Signal producer Operations
和`Signal`一样，`SignalProducer`的操作也被定义成`curried free function`。
例如用来在pipline中插入副作用的`on`操作（为了可读性省略了一些函数参数）：
```swift
func on<T, E>(started: (() -> ())? = nil, ...)
             (producer: SignalProducer<T, E>) -> SignalProducer<T, E> {
    ...
}
```

`pipe-forward`运算符也为`SignalProducer`进行了重载：
```swift
public func |> <T, E, X>(producer: SignalProducer<T, E>,
                         transform: SignalProducer<T, E> -> X) -> X {
    return transform(producer)
}
```
所以，我们可以像这样为`SignalProducer`创建pipeline：
```swift
signalProducer
    |> on(started: {
        print("Signal has started")
    })
    |> start(next: {
        print("Next received: \($0)")
    })
```

# Applying Signal Operations to Signal Producers
`SignalProducer`通过`lift`方法来复用`Signal`上的操作，例如`map`、`filter`等。`lift`会将`Signal`的操作应用在`SignalProducer`通过`start`创建的信号上。
```swift
let signalProducer = createSignalProducer()

let mapping: Signal<String, NoError> -> Signal<Int, NoError> = map({
    string in
    return count(string)
})
let mappedProducer = signalProducer.lift(mapping)

mappedProducer
    |> start(next: {
        print("Next received: \($0)")
    })
```
`pipe-forward`运算符也为`SignalProducer`做了重载：
```swift
public func |><T, E, U, F>(producer: SignalProducer<T, E>, 
                          transform: Signal<T, E> -> Signal<U, F>) -> SignalProducer<U, F> {
    return producer.lift(transform)
}
```
因此，`Signal`的操作都可以直接通过`pipe-forward`应用在`SignalProducer`上：
```swift
signalProducer
    |> on(started: {
        print("Signal has started")
    })
    |> map { count($0) }
    |> start(next: {
        print("Next received: \($0)")
    })
```