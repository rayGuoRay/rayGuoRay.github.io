---
title: Handler 消息机制源码分析（三）
date: 2016-08-18 19:34:56
categories: 源码分析
tags: [Android, Handler]
keywords: Android, Handler
comments: true
---

在前面两篇文章中我们介绍了 Hanlder 的构造方法与 Looper 的初始化。本篇中我们将对主线程 Looper 与 Handler 消息循环的工作方式进行分析。

*前文链接如下所示*
- [《Handler 消息机制源码分析（一）》](http://wizardiy.com/2016/06/17/Handler%20%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%80%EF%BC%89/)
- [《Handler 消息机制源码分析（二）》](http://wizardiy.com/2016/06/18/Handler%20%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%BA%8C%EF%BC%89/)

### 主线程的Looper初始化
主线程的 looper 的初始化是在 Looper.prepareMainLooper 中进行初始化的。而这个方法被调用位置在 ActivityThread 中 main 方法中其代码引用如下：

    Looper.prepareMainLopper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    if (false) {
        Looper.myLooper().setMessag
    }
    Looper.loop();


而 ActivityThread 的 main 方法是 Android 应用程序主线程的入口，所以我们可以理解为：主线程的 Looper 在应用启动时已经初始化过了。

### Handler 的消息发送
我们在创建好 Ｈandler 后，使用 Handler 发送消息一般调用根据其方法调用有:

- sendMessage()
- sendEmptyMessage()
- sendMessageAtTime()
- sendMessageDelayed()

其中主要区别在于如下源码展示:

    public final boolean sendMessage(Message msg) {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendEmptyMessage(int what) {
        return sendEmptyMessageDelayed(what, 0);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }


    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

可以看到，当发送 EmptyMessage 时，会在 Handler 中使用 Message.obtain() 来构造一个 Message ，而如果发送无延时消息时，会设置延时时间为0，最后会发送都会调用 sendMessageAtTime() 方法。所有的消息都会通过 enqueMessage 发送出去。

*注意： Hanlder 中的 enqueMessage 中会调用其中 messageQueue.enqueMessage(msg, uptimeMillis) ,但是 Handler 中的 MessageQueue 对象是从 Looper 中 get 到的，所以，MessageQueue 耦合的还是 Looper*

### MessageQueue 的消息遍历

    if (p == null || when == 0 || when < p.when) {
        // New head, wake up the event queue if blocked.
        msg.next = p;
        mMessages = msg;
        needWake = mBlocked;
    } else {
        // Inserted within the middle of the queue.  Usually we don't have to wake
        // up the event queue unless there is a barrier at the head of the queue
        // and the message is the earliest asynchronous message in the queue.
        needWake = mBlocked && p.target == null && msg.isAsynchronous();
        Message prev;
        for (;;) {
            prev = p;
            p = p.next;
            if (p == null || when < p.when) {
                break;
            }
            if (needWake && p.isAsynchronous()) {
                needWake = false;
            }
        }
        msg.next = p; // invariant: p == prev.next
        prev.next = msg;
    }
    if (needWake) {
        nativeWake(mPtr);
    }

如上图中代码中所示：其中主要操作是将新加入的 Message 与当前的 MessageQueue 中的头进行比较，如果当前有新消息进入并且触发时间为0或小于当前头节点的时间，如果队列阻塞，则唤醒线程处理。如果新消息触发时间长于队列中的消息时间，则按时间排序，插入到队列中。如果唤醒线程，则调用 nativeWake 函数进行 JNI 方法调用发送消息。

### 消息循环处理与回调
Looper 中的 loop() 方法会拿到与 looper 耦合的 MessageQueue 对象，由 MessageQueue 调用 next() 方法，如果 meesage 对象不为空，则调用 msg.target.dispatchMessage(msg) 回调到 Handler 中。

    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

其中，else 语句中的 handlerMessage 最终会回调回当时创建的 Handler 对应的 handMessage 方法中。

*在回调中，我们注意到除了 handleMessage 的回调，还有可以注册 message 的 CallBack 或者 Handler的 callback 来回调。message 的 Callback 需要在 message 创建时传入一个 Runnable 对象。而 Handler的 Callback 与 handleMessage 方法就目前而言使用上没感觉有什么区别。
希望有了解的同学能对这个问题留言说明下，谢谢～！*

##### 写到最后
>Handler的基础分析就算完成了，后续有时间会做其他相关的源码分析。
不足之处，请大家批评指导。
