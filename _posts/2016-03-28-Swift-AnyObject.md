---
layout: post
title: "AnyObject in Swift"
date: 2016-03-28 08:00:00 +0800
category: Tech
tags: Swift, AnyObject
---

`AnyObject`相当于Swift的基类, 其实在实际编程过程中, 很少会需要把一个对象声明成`AnyObject`, 而且这货也没啥方法, 有啥好讲的呢. 

在我们的App中, `AnyObject`主要用在网络请求回来的数据上, 请求回来的数据要拿来初始化model, 这时候一般都是`AnyObject`类型. 那有什么问题呢? 看这段代码:

```swift
let x: AnyObject = UIView()
let y = x.stringValue                   // 1. 不报错, y的类型是String!
x.testabc                               // 2. 编译报错: value of type 'AnyObject' has no member 'testabc'
x.backgroundColor                       // 3. 编译报错: ambiguous use of 'backgroundColor'
x.setTitle?("", forSegmentAtIndex: 0)   // 4. 不报错
x.setTitle("", forSegmentAtIndex: 0)    // 5. 运行时报错: unrecognized selector
```

**为什么1, 4不报错, 2, 3, 5报错呢?**

我们来看一下苹果官方对`AnyObject`的说明:

`The protocol to which all classes implicitly conform. When used as a concrete type, all known @objc methods and properties are available, as implicitly-unwrapped-optional methods and properties respectively, on each instance of AnyObject.`

所有的类实际上都是实现了这个`protocol`, 当一个变量声明成AnyObject的时候, 那么所有可以被这个变量访问的类中, 被标记为@objc的方法和被标记为@objc的属性对这个变量都是可见的, 而且是隐式解包的optional类型, 也就是`!`.

这样就可以解释上述代码中的问题了:
- `1`中`stringValue`是`NSNumber`的property, `x.stringValue`的返回值类型是`String!`, 因为对于AnyObject来讲, `stringValue`是隐式解包的optional. 如果`x`的实际类型是`NSNumber`的话, 返回值是正常的`String!`; 其它情况下, 返回值是`nil`
- `2`中的`testabc`并不是AnyObject的子类中`@objc`的属性, 编译的时候找不到.
- `3`中的`backgroundColor`虽然是`@objc`的属性, 但是因为在UIView和CALayer上都有定义, 编译器不知道用哪个.
- `4`采用了`optional chain`的调用方式, 如果方法不存在, 直接返回`nil`, 不会报错.
- `5`是直接调用该方法, 但是运行时, 系统查询`UIView`并没有这个方法, 抛出异常