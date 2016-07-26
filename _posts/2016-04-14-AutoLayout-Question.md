---
layout: post
title: "Interesting Part of AutoLayout"
date: 2016-04-14 17:04:00 +0800
categories: iOS, AutoLayout
---

当我们给view做约束的时候, 系统算leading, trailing的值是多少呢? 或者说是参考什么呢?

比方说, 我们有v, v1, v2这样三个view. 考虑两种情况:

1. v是v1和v2的父view, `v2.leading = v1.trailing`, 那么这个时候, 在计算这个约束的时候, trailing的值是多少呢?

2. 如果v是v1的父view, 而v1在作为v2的父view, 约束`v2.leading = v1.trailing`, 这个时候, 计算式trailing的值是多少呢?

实践证明, 计算约束时的坐标系是已参与该约束的views的公共最低父view的坐标系作为基准的.

那么知道这个有什么用呢? 其实也没啥用... 
