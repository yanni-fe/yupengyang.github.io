---
layout: post
title: "How to use private pods"
date: 2016-05-17 16:01:00 +0800
categories: iOS, CocoaPods
---

#How to use private pods

iOS开发目前大部分都是用cocoaPods来做依赖管理的. 而且大部分都是用来管理引入的第三方库. 但是如果我们要创建公司内部私有的共用库, 应该怎么做呢? 可以使用cocoaPods的private pods功能.

大概步骤如下:

1. 创建一个Private Spec Repo的gitRepo
2. 把这个gitRepo添加到cocoaPods的repo list里面
3. 创建你私有项目的`podspec`文件
4. 向这个私有的cocoaPods repo里面添加你的私有项目的`podspec`文件
5. 在你工程的`Podfile`里面添加这个私有的cocoaPods repo, 并引用你的私有项目.

举例: 假设现在需要创建一个叫`Utils`的私有pod, 并且在项目`Demo`中使用它.

### 1. 创建Private Spec Repo的gitRepo

这个其实很简单, 就是在你的git server上创建一个空的project就可以了. 比如在Github上创建一个repo.

### 2. 把这个gitRepo添加到cocoaPods的repo list里面

运行命令:

```shell
$ pod repo add REPO_NAME SOURCE_URL
```

`REPO_NAME`是自己起的这个repo的名字. `SOURCE_URL`是你第一步创建的git的地址, 推荐`https`的地址. 比如`https://github.com/yupengyang/repo.git`.

然后进入你的cocoaPods的repo目录里面(默认是~/.cocoapods/repos/), 会看到除了`master`之外多了一个`REPO_NAME`的文件夹, 运行如下命令:

```shell
$ cd ~/.cocoapods/repos/REPO_NAME
$ pod repo lint .
```

没有问题说明你的private spec repo已经创建完毕.

### 3. 创建你私有项目的`podspec`文件

我们需要创建`Utils`库的`podspec`文件, 在Utils的根目录下运行

```shell
$ pod spec create SPEC_NAME(我们的例子中这里是Utils)
```

然后cocoaPods会创建一个`Utils.podspec`的模板文件, 我们只需要按照需求修改就可以了. 具体`podspec`的语法参见[这里](https://guides.cocoapods.org/syntax/podspec.html).

写完之后可以运行`$ pod spec lint .`来检查`podspec`是否正确

### 4. 向私有的cocoaPods repo里面添加你的私有项目的`podspec`文件

运行命令:

```shell
$ pod repo push REPO_NAME SPEC_NAME.podspec
```

这样你的私有项目就放到了你私有的cocoaPods的repo里面.

### 5. 使用

在你项目的`Podfile`里面添加私有的cocoaPods的repo. 如果还引用了开源的第三方库, 还要在显示的把cocoaPods默认的repo加进来.

```ruby
source 'SOURCE_URL(https://github.com/yupengyang/repo.git)'
source 'https://github.com/CocoaPods/Specs.git'
```

然后就正常使用`Utils`的这个pod就可以了

```ruby
pod 'Utils'
```

over.

