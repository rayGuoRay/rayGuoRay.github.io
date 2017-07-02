---
title: Handler 消息机制源码分析（一）
date: 2016-06-17 18:56:56
categories: 源码分析
tags: [源码分析, Android, Handler]
keywords: 源码分析, Android, Handler
comments: true
---

一直在用 Handler,但却一直因为不能准确表述其工作原理而耿耿于怀，现在正好有时间，自己决定初步梳理下。如果有整理不足或不准确的地方，请大家批评指正。

相信大家对于主线程中创建并使用 Handler 一点都不陌生了。如下图所示：（声明为 static 的静态类原因是避免 Handler 作为内部类，避免隐式持有外部引用导致的内存泄漏问题。）

![](http://upload-images.jianshu.io/upload_images/1489253-325849abb2712457.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

或许大家也有碰到过类似于下面的声明方法:
- Handler mainHandler = newHandler(Looper.getMainLooper());</p>

但是阅读 Handler 源码发现，Handler 提供了多种构造方法以供使用:

1. public Handler() { this(null,false); }
2. public Handler(Callback callback) { this(callback,false); }
3. public Handler(Looper looper) { this(looper,null,false); }
4. public Handler(Looper looper, Callback callback) { this(looper, callback,false);}
5. public Handler(boolean async) {this(null, async);}
6. public Handler(Callback callback, boolean async) {}
7. public Handler(Looper looper, Callback callback,booleanasync) {}

可以看到，如果未传入最常见的参数为空的构造方法其实引用的序号为6的构造方法。其实大家应该也有发现：Handler 如果不显式传入 Looper 创建对象的话，最后都会调用序号为6的构造方法，而如果显示传入 Looper 的话，最后都会调用序号为7的构造方法。

而构造方法6与构造方法7的区别是什么呢？附上源码大家应该就比较清楚了

![](http://upload-images.jianshu.io/upload_images/1489253-bcbc1d99a5675e43.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，唯一的区别或许就是mLooper的获取方式了，而在构造方法6中， 如果mLooper对象为空的时候，会抛出一个异常，称在线程中创建handler时没用调用Looper.prepare()方法。我想这也许就是大多数时候在子线程中调用构造方法1，时会抛出异常的原因了。</p><p>至于为何会抛出异常，请期待对Looper类的分析吧～～～😊

*ps：或许有哪位小伙伴能回答下FIND_POTENTIAL_LEAKS这个参数有什么意义呢？*

相关文章链接：
- [Handler 消息机制源码分析(二)]()
- [Handler 消息机制源码分析(三)]()
