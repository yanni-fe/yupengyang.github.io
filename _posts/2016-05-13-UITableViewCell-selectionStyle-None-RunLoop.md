---
layout: post
title: "UITableViewCell的selectionStyle置为None引发的..."
date: 2016-05-13 15:29:00 +0800
categories: iOS, RunLoop
---

今天在工程里面遇到一个问题, 一个页面使用了TableView, 有一个cell是说明性的, 点击后present一个说明的VC. 根据产品要求, 这个cell不要点击效果. 于是`selectionStyle`设置成了`.None`.

但是present的时候出现了一个问题, 点击了cell, 什么都没有发生...再点一下, 才会正常弹出VC. 一开始并没有在意这个问题, 以为是手指没有点上, 但是后来发现每次都这样. 如果不点第二下, 会等很久才弹出VC. 调试无果. `presentViewController`的代码已经运行, 但是界面没有弹出来. 但是有时候, 点击之后过几秒就弹出来了. 当时和同事想了半天, 觉得可能是RunLoop的问题. 因为只要我在屏幕上随便点击一下, 就会再次弹出来, 说明在这之前`RunLoop`可能进入了休眠状态. 于是在`presentViewController`的代码后面加上了一个`dispatch_async..`的空的block来验证猜想, 发现问题解决. 但是不明白为什么会这样. 

Google之, 在SO上找到一篇[帖子](http://stackoverflow.com/questions/21075540/presentviewcontrolleranimatedyes-view-will-not-appear-until-user-taps-again). 可以算是苹果的一个bug. 当`UITableViewCell`的`selectionStyle`设置为`.None`的时候, 并且要去`presentViewController`的时候, 就会触发这个bug, 但是在`UINavigationController`上`pushViewController`就没有问题. 也挺奇怪的.