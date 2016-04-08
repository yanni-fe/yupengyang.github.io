---
layout: post
title: "@objc"
date: 2016-04-08 14:24:00 +0800
categories: swift, iOS
---

前段时间在程序里遇到一个bug, 实例代码如下:

```swift
class B {
    let content: [Any] // 因为考虑可能会接受tuple, 所以用了Any
    init(content: [Any]) {
        self.content = content
    }
}
let obj = [NSObject()]
B(content: obj.map { $0 })  // 1. 不报错
B(content: obj) // 2. 运行时异常: array cannot be bridged from Objective-C
```

1不报错应该容易理解, 因为Swift经过类型推断, `map`传入参数的返回值类型为`Any`, `map`函数的返回值为`[Any]`, 类型正确. 2错误提示是array不能从OC被桥接, 发生在我把`[NSObject]`类型传递给`[Any]`的时候. 难道是Cocoa的类和Swift的Protocol之间不能交互? 于是做了下面的实验:

```swift
protocol P {
    
}
class PImpl: P {
    
}
let pi = [PImpl()]
let p: [P] = pi // 运行时异常: array cannot be bridged from Objective-C
let op: [AnyObject] = obj       // 不报错
```

`Protocol P`和`Any`出现了同样的错误, 但是`AnyObject`却没有问题. 那`P`, `Any`和`AnyObject`的区别是什么呢? 我们看下定义:

```swift
// P: 
protocol P

// Any:
public typealias Any = protocol<>

// AnyObject:
@objc public protocol AnyObject
```

忽略`public`之后, 唯一的区别就是`AnyObject`多了一个`@objc`. 那么如果给`P`加上`@objc`会不会就不会报错了?

```swift
@objc protocol PAgain {
    
}
class PImplAgain: PAgain {
    
}
let pia = [PImplAgain()]
let pa: [PAgain] = pia    // 不报错!
```

看来果然是`@objc`的问题. 那这个货到底干了啥呢? 我们看下官方文档的解释:

>Apply this attribute to any declaration that can be represented in Objective-C—for example, non-nested classes, protocols, nongeneric enumerations (constrained to integer raw-value types), properties and methods (including getters and setters) of classes and protocols, initializers, deinitializers, and subscripts. The objc attribute tells the compiler that a declaration is available to use in Objective-C code.

>Classes marked with the objc attribute must inherit from a class defined in Objective-C. If you apply the objc attribute to a class or protocol, it’s implicitly applied to the Objective-C compatible members of that class or protocol. The compiler also implicitly adds the objc attribute to a class that inherits from another class marked with the objc attribute or a class defined in Objective-C. Protocols marked with the objc attribute can’t inherit from protocols that aren’t.

`@objc`的意思是, 所有用这个属性标识的thing都能在OC里使用. 编译器应该会对这些thing做一些处理, 使得OC能够使用. 基于这些实验, 对上述问题原因的猜测解释: 数组在runtime底层的转换工作实际上都是OC再处理的. 如果一个Swift的Protocol没有`@objc`标记, 那么runtime处理是OC就不认识它, 就没办法处理. 至于为啥没有`@objc`标记的Protocol就不能处理, 我猜是因为Swift的Protocol可以被Struct和Enum继承的, 但是Struct和Enum是不能出现在NSArray中的.

> 个人理解, 出错请指正

更详细的实现可以参考so上一个答案: [link](http://stackoverflow.com/questions/35893517/is-it-possible-to-replicate-swifts-automatic-numeric-value-bridging-to-foundatio)
