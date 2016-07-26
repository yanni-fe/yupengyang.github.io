---
layout: post
title: "Swift 初始化方法的一个tip"
date: 2016-07-26 13:29:00 +0800
categories: iOS, Swift
---

先来看一段code

```swift
class A {
    init() {
        foo()
    }
    func foo {
        print("A")
    }
}

class B: A {
    var b: Int
    init(b: Int) {
        self.b = b
        super.init()
    }
}
let b = B(b: 1)
```

这是一个基础的swift类继承, 这里有一个点, B的初始化方法中对`b`的初始化工作一定要在`super.init()`调用之前. 如果不这样就会报错.

这个规则都知道. 但是为啥要这样?

看下面这个例子

```swift
class A {
    init() {
        foo()
    }
    func foo {
        print("A")
    }
}

class B: A {
    var b: Int
    init(b: Int) {
        self.b = b
        super.init()
    }
    override func foo() {
        print("\(self.b)")
    }
}
let b = B(b: 1)
// 输出结果为: 1\n
```

在`A`的初始化方法里面调用了`foo`函数, 而`B`重写了`foo`方法, 在`B`初始化的时候实际调用的是子类`B`的`foo`方法. 而在`B.foo`中使用了`B.b`. 如果`b`的初始化在`super.init`之后, 那么这个时候`foo`中的`b`是没有初始化的, 就会出问题. 

可能这就是为什么初始化的时候需要先初始化property才能再调用`super`初始化方法的原因.
