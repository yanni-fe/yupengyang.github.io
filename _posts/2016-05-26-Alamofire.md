---
layout: post
title: "Alamofire原理简析"
date: 2016-05-26 21:51:00 +0800
categories: iOS, Swift, Alamofire
---

#Alamofire流程

`Manager`维护了一个session和session的delegate(`SessionDelegate`), 

1. 当用户调用`request(...)`方法
	
	Manager会创建一个task(`NSURLSessionTask`), 然后用这个task去初始化一个request(`Request`). 

	在request的初始化方法里面, 会创建一个subDelegate(`TaskDelegate`), 并作为request的property存储.

	subDelegate会创建一个串行的queue(`NSOperationQueue`), 不过这个queue在初始化的时候是阻塞的(`queue.suspended = true`). 

	创建好request之后, session.delegate会用task作为key, 把subDelegate作为value存下来. 

	调用request.resume()启动任务. (在后台运行)

2. 用户调用request的response....方法 (因为这个方法是在`1.`之后同步调用, 所以一定发生在request请求结果回来之前)

	会向subDelegate的queue里添加一个block, block里面会创建一个`Response`对象并在main_queue上调用用户指定的completionHandler. 这里要注意的是, 这个block不会立即执行, 因为subDelegate的queue是阻塞状态.
	
3. 请求结果回来

	先调用manager.session.delegate, 这个delegate会查找task对应的subDelegate, 并把delegate事件转发给subDelegate执行.
	
	当subDelegate的`URLSession:task:didCompleteWithError:)`回调完成之后, subDelegate会把它的queue关闭阻塞状态(`queue.suspended = false`). 
	
	在`2.`中添加的block就会开始按顺序执行.
	
	





