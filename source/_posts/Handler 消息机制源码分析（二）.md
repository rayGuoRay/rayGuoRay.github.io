---
title: Handler 消息机制源码分析（二）
date: 2016-06-18 17:16:56
categories: 源码分析
tags: [源码分析, Android, Handler]
keywords: 源码分析, Android, Handler
comments: true
---

在上一片文章中[Handler 消息机制源码分析（一）](http://wizardiy.com/2016/06/17/Handler%20%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%80%EF%BC%89/)中我们分析到Handler在创建时会绑定一个 Looper 对象，而Looper 如果用户不进行显式传入，会在 Handler 中调用 Looper.myLooper() 来创建一个 Looper 对象。

那什么是 Looper 呢？ Android 中对于 Looper 类的注释中是这样描述的：“Class used to run a message loop for a thread”。大概意思是指这个类是用来为一个线程做消息循环。而 Looper 类本身被声明为 final 类型的，同时它的构造方法也被声明为 final 类型。所以我们如果要获取其中的 Looper 实例或者使用 Looper 的方法需要使用其中提供的两种静态方法：

![](http://upload-images.jianshu.io/upload_images/1489253-6a6158c9a3c87f21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，无论是 getMainLooper() 方法还是 myLooper() 方法，获取的 Looper 都是直接返回已经初始化的对象。那它们的初始化在哪操作的呢？如下图所示：

![](http://upload-images.jianshu.io/upload_images/1489253-15a623d946d67366.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，主线程 Looper 和子线程 Looper 创建的区别在于一个 quitAllowed 变量（主线程为 false , 子线程为 true ）。而最终这个变量会传入到 Looper 构造方法中用于构造 MessageQueue 。而 Looper 中的线程绑定的是当前线程。如下图所示：

![](http://upload-images.jianshu.io/upload_images/1489253-8846ed6a2eeca544.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样的话，结合上一篇，我们可以发现，直接在子线程中使用 Handler handler = new Handler(),会调用 Looper.myLooper() 方法获取 Looper 对象，在这之前并没有调用 Looper.prepare() 方法进行初始化，所以 sThreadLocal.get() 中会返回 null ，进而报错。

那大家或许会问，主线程中的 mainLooper 是何时进行初始化的呢，为什么主线程中直接 new Handler() 不会报错呢？其实这个主线程 Handler 在应用启动时就已经进行了初始化，具体启动时机需要在后面继续分析呢～😊

＊ps：后续文章传送门：＊

- [Handler 消息机制源码分析（三）](http://wizardiy.com/2016/08/18/Handler%20%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%89%EF%BC%89/)
