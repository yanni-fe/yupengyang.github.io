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

首先解释关于`optional`部分的问题. 我们都知道如果一个函数的返回值是`optional`, 例如`Int?`, 我们`let i: Int = 1; return i`是没有问题的. 这里的`i`的类型是`Int`而不是`Int?`, 但是编译时不报错, 运行时也没有问题. 这说明Swift对于`optional`进行了优化, 至于是编译时还是运行时呢, 是编译时的. 例如下面的代码:

``` swift
func f(a: String?) -> String? {
    let x = "hehe"
    return x
}
let a = "yupengyang"
f(a)
```

在命令行用`swiftc -emit-sil your_swfit_file`就可以看到编译后的中间语言. 先看调用时, 函数参数是`optional`而传入的不是`optional`的情况

```swift
let a = "yupengyang"
f(a)
// 这两行对应的中间语言大概如下:
%12 = global_addr @_Tv4test1aSS : $*String      // users: %19, %21
  // function_ref Swift.String.init (_builtinStringLiteral : Builtin.RawPointer, byteSize : Builtin.Word, isASCII : Builtin.Int1) -> Swift.String
  %13 = function_ref @_TFSSCfT21_builtinStringLiteralBp8byteSizeBw7isASCIIBi1__SS : $@convention(thin) (Builtin.RawPointer, Builtin.Word, Builtin.Int1, @thin String.Type) -> @owned String // user: %18
  %14 = metatype $@thin String.Type               // user: %18
  %15 = string_literal utf8 "yupengyang"          // user: %18
  %16 = integer_literal $Builtin.Word, 10         // user: %18
  %17 = integer_literal $Builtin.Int1, -1         // user: %18
  %18 = apply %13(%15, %16, %17, %14) : $@convention(thin) (Builtin.RawPointer, Builtin.Word, Builtin.Int1, @thin String.Type) -> @owned String // user: %19
  store %18 to %12 : $*String                     // id: %19
  //------------到此为止创建了String "yupengyang"
  
  // function_ref test.f (Swift.Optional<Swift.String>) -> Swift.Optional<Swift.String>
  %20 = function_ref @_TF4test1fFGSqSS_GSqSS_ : $@convention(thin) (@owned Optional<String>) -> @owned Optional<String> // user: %24
  %21 = load %12 : $*String                       // users: %22, %23
  retain_value %21 : $String                      // id: %22
  %23 = enum $Optional<String>, #Optional.Some!enumelt.1, %21 : $String // user: %24
  //------------上面这一句就是把String转成了Optional<String>了
  
  %24 = apply %20(%23) : $@convention(thin) (@owned Optional<String>) -> @owned Optional<String> // user: %25
```

在看函数定义的返回值类型是`optional`, 而实际返回的类型不是`optional`的情况, 函数`f`对应的中间代码摘取如下

``` swift
debug_value %0 : $Optional<String>, let, name "a", argno 1 // id: %1
  // function_ref Swift.String.init (_builtinStringLiteral : Builtin.RawPointer, byteSize : Builtin.Word, isASCII : Builtin.Int1) -> Swift.String
  %2 = function_ref @_TFSSCfT21_builtinStringLiteralBp8byteSizeBw7isASCIIBi1__SS : $@convention(thin) (Builtin.RawPointer, Builtin.Word, Builtin.Int1, @thin String.Type) -> @owned String // user: %7
  %3 = metatype $@thin String.Type                // user: %7
  %4 = string_literal utf8 "hehe"                 // user: %7
  %5 = integer_literal $Builtin.Word, 4           // user: %7
  %6 = integer_literal $Builtin.Int1, -1          // user: %7
  %7 = apply %2(%4, %5, %6, %3) : $@convention(thin) (Builtin.RawPointer, Builtin.Word, Builtin.Int1, @thin String.Type) -> @owned String // users: %8, %9
  debug_value %7 : $String, let, name "x"         // id: %8
  //----以上部分创建了变量x, 也就是我们要返回的String
  
  %9 = enum $Optional<String>, #Optional.Some!enumelt.1, %7 : $String // user: %11
  //----这一句把String转成了Optional<String>了
  
  release_value %0 : $Optional<String>            // id: %10
  return %9 : $Optional<String>                   // id: %11
  //----返回Optional<String>
```

``` swift
func f(a: String?) -> String? {
    let x = "hehe"
    return x
}
let a = "yupengyang"
f(a)
```

在命令行用`swiftc -emit-sil your_swfit_file`就可以看到编译后的中间语言. 先看调用时, 函数参数是`optional`而传入的不是`optional`的情况

```swift
let a = "yupengyang"
f(a)
// 这两行对应的中间语言大概如下:
%12 = global_addr @_Tv4test1aSS : $*String      // users: %19, %21
  // function_ref Swift.String.init (_builtinStringLiteral : Builtin.RawPointer, byteSize : Builtin.Word, isASCII : Builtin.Int1) -> Swift.String
  %13 = function_ref @_TFSSCfT21_builtinStringLiteralBp8byteSizeBw7isASCIIBi1__SS : $@convention(thin) (Builtin.RawPointer, Builtin.Word, Builtin.Int1, @thin String.Type) -> @owned String // user: %18
  %14 = metatype $@thin String.Type               // user: %18
  %15 = string_literal utf8 "yupengyang"          // user: %18
  %16 = integer_literal $Builtin.Word, 10         // user: %18
  %17 = integer_literal $Builtin.Int1, -1         // user: %18
  %18 = apply %13(%15, %16, %17, %14) : $@convention(thin) (Builtin.RawPointer, Builtin.Word, Builtin.Int1, @thin String.Type) -> @owned String // user: %19
  store %18 to %12 : $*String                     // id: %19
  //------------到此为止创建了String "yupengyang"
  
  // function_ref test.f (Swift.Optional<Swift.String>) -> Swift.Optional<Swift.String>
  %20 = function_ref @_TF4test1fFGSqSS_GSqSS_ : $@convention(thin) (@owned Optional<String>) -> @owned Optional<String> // user: %24
  %21 = load %12 : $*String                       // users: %22, %23
  retain_value %21 : $String                      // id: %22
  %23 = enum $Optional<String>, #Optional.Some!enumelt.1, %21 : $String // user: %24
  //------------上面这一句就是把String转成了Optional<String>了
  
  %24 = apply %20(%23) : $@convention(thin) (@owned Optional<String>) -> @owned Optional<String> // user: %25
```

在看函数定义的返回值类型是`optional`, 而实际返回的类型不是`optional`的情况, 函数`f`对应的中间代码摘取如下

``` swift
debug_value %0 : $Optional<String>, let, name "a", argno 1 // id: %1
  // function_ref Swift.String.init (_builtinStringLiteral : Builtin.RawPointer, byteSize : Builtin.Word, isASCII : Builtin.Int1) -> Swift.String
  %2 = function_ref @_TFSSCfT21_builtinStringLiteralBp8byteSizeBw7isASCIIBi1__SS : $@convention(thin) (Builtin.RawPointer, Builtin.Word, Builtin.Int1, @thin String.Type) -> @owned String // user: %7
  %3 = metatype $@thin String.Type                // user: %7
  %4 = string_literal utf8 "hehe"                 // user: %7
  %5 = integer_literal $Builtin.Word, 4           // user: %7
  %6 = integer_literal $Builtin.Int1, -1          // user: %7
  %7 = apply %2(%4, %5, %6, %3) : $@convention(thin) (Builtin.RawPointer, Builtin.Word, Builtin.Int1, @thin String.Type) -> @owned String // users: %8, %9
  debug_value %7 : $String, let, name "x"         // id: %8
  //----以上部分创建了变量x, 也就是我们要返回的String
  
  %9 = enum $Optional<String>, #Optional.Some!enumelt.1, %7 : $String // user: %11
  //----这一句把String转成了Optional<String>了
  
  release_value %0 : $Optional<String>            // id: %10
  return %9 : $Optional<String>                   // id: %11
  //----返回Optional<String>
```

由此可见, swift是在编译时期就做了这个转换的.

下面回头我们最开始的例子, 参数是闭包的情况. 上面的代码中, `test`函数接收的参数是`B -> A?`的一个闭包. 我们可以传入`B? -> A`的闭包, 但是如果`test`接收的是`B? -> A`的闭包, 我们不能传入`B -> A`的闭包, 如下:

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
