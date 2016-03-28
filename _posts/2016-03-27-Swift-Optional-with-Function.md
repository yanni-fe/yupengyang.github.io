---
layout: post
title: "Optional和Function"
date: 2016-03-27 10:00:00 +0800
categories: Tech
tags: Swift, Optional, Function
---

在InfoQ上看唐巧推荐了一篇关于Swift中monad的[文章][0]

看了之后对于`Optional`类型能用`map`和`flatMap`很是兴奋, 于是拉上同事开始对项目中大量的`if let ... else`进行重写, 然后在重写中发现了一个有趣的问题. 先看下optional中`map`和`flatMap`的函数定义:

``` swift
/// If `self == nil`, returns `nil`.  Otherwise, returns `f(self!)`.
@warn_unused_result
public func map<U>(@noescape f: (Wrapped) throws -> U) rethrows -> U?

/// Returns `nil` if `self` is nil, `f(self!)` otherwise.
@warn_unused_result
public func flatMap<U>(@noescape f: (Wrapped) throws -> U?) rethrows -> U?
```

区别就在接收的参数f的定义不同. `map`的参数f的返回值不是`Optional`, 而`flatMap`的参数f的返回值是`Optional`. 但是看下面代码:

``` swift
let f: Int -> Int = { $0 }
let s: Int? = 1
let res = s.flatMap(f)  // 没有报错!
```

什么情况, `flatMap`接收的不应该是`Int -> Int?`类型么? 继续往下看:

``` swift
if f is (Int -> Int?) { // 编译器warning: 'is' test is always true
    print("hehe")
} else {
    print("no")         // 运行时 进入此分支, 打印no
}
```

编译时警告和运行结果不一样...我在so上问了这个问题: [question][1] 等待答案. 个人倾向这个是bug... Anyway, 先不管这个问题, 看下面的代码:

``` swift
class A {}
class B: A {}

func f(a: A?) -> B { return B() }

func test(f: B -> A?) {
	let b = B()
    print(f(b))
}
test(f)
```

接受一个`B -> A?`的参数, 结果传入一个`A? -> B`的参数都没问题. 

首先解释关于`optional`部分的问题. 我们都知道如果一个函数的返回值是`optional`, 例如`Int?`, 我们`let i: Int = 1; return i`是没有问题的. 这里的`i`的类型是`Int`而不是`Int?`, 但是编译时不报错, 运行时也没有问题. 这说明Swift对于`optional`进行了优化, 至于是编译时还是运行时呢, 是编译时的, 你可以写一个方法测试下, 在命令行用`swiftc -emit-sil your_swfit_file`就可以看到编译后的中间语言. 这个可以解释`optional`的问题. 函数参数是`optional`的时候类似. 

但是要注意的是, 上述代码中, `test`函数接收的参数是`B -> A?`的一个闭包. 我们可以传入`B? -> A`的闭包, 但是如果`test`接收的是`B? -> A`的闭包, 我们不能传入`B -> A`的闭包, 如下:

``` swift
class A {}
class B: A {}

func test1(f: B? -> A?) {
	let b: B? = nil
    f(b)
}
```

下面解释下原因. 看`test1`函数, 传入的闭包实际上是要在`test1`中被调用的, 在上面的代码`test1`函数里面, 变量`b`的类型是`B?`, 闭包`f`的类型是`B? -> A?`, 这个时候调用`f(b)`, 编译器进行类型检查, 发现`b`的类型和闭包`f`的入参类型是一致的, 没有问题. 如果我们传入的`f`的类型是`B -> A?`, 那么调用`f(b)`的时候, 编译器会发现, `Optional<B>`类型的`b`要传入接收`B`类型的闭包`f`, 编译器不能从`Optional<B>`转成`B`, 所以编译时报错.

[0]: http://www.mokacoding.com/blog/functor-applicative-monads-in-pictures/
[1]: http://stackoverflow.com/questions/36173726/swift-use-is-for-function-type-compiler-behavior-is-different-with-runtime
