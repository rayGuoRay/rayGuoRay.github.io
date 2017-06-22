##### 写在前面的话
>简书支持Markdown语法，从这篇文章开始，后面的文章都会改成Markdown编辑，让我们一起来学着使用Markdown吧。：）

在前面两篇文章中
- [《Android Handler消息机制源码分析（一）》](http://www.jianshu.com/p/2b055625b400)
- [《Android Handler消息机制源码分析（二）》](http://www.jianshu.com/p/3be5cca5c518)

中我们介绍了Hanlder的构造方法与Looper的初始化。本篇中我们将对剩余部分进行剩余部分的分析。
***
##### 主线程的Looper初始化
主线程的looper的初始化是在Looper.prepareMainLooper中进行初始化的。而这个方法被调用位置在ActivityThread中main方法中其代码引用如下：

`Looper.prepareMainLopper();
 ActivityThread thread = new ActivityThread();
 thread.attach(false);
 if (false) {
    Looper.myLooper().setMessag
 }
 Looper.loop();
`

而ActivityThread的main方法是Android应用程序主线程的入口，所以我们可以理解为：主线程的Looper在应用启动时已经初始化过了。

##### Handler的消息发送
我们在创建好Ｈandler后，使用Handler发送消息一般调用根据其方法调用有:
sendMessage()
sendEmptyMessage()
sendMessageAtTime()
sendMessageDelayed()
其中主要区别在于如下源码展示:

`public final boolean sendMessage(Message msg) {
    return sendMessageDelayed(msg, 0);
 }`

 `public final boolean sendEmptyMessage(int what) {
    return sendEmptyMessageDelayed(what, 0);
 }`

 `public final boolean sendMessageDelayed(Message msg, long delayMillis)
 {
     if (delayMillis < 0) {
         delayMillis = 0;
     }
     return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
 }
 `

 `public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }
 `   

 `public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
`
可以看到，当发送EmptyMessage时，会在Handler中使用Message.obtain()来构造一个Message，而如果发送无延时消息时，会设置延时时间为0，最后会发送都会调用sendMessageAtTime()方法。所有的消息都会通过enqueMessage发送出去。

*注意：Hanlder中的enqueMessage中会调用其中messageQueue.enqueMessage(msg, uptimeMillis),但是Handler中的MessageQueue对象是从Looper中get到的，所以，MessageQueue耦合的还是Looper*

##### MessageQueue的消息遍历
`if (p == null || when == 0 || when < p.when) {
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
            }`

如上图中代码中所示：其中主要操作是将新加入的Message与当前的MessageQueue中的头进行比较，如果当前有新消息进入并且触发时间为0或小于当前头节点的时间，如果队列阻塞，则唤醒线程处理。如果新消息触发时间长于队列中的消息时间，则按时间排序，插入到队列中。如果唤醒线程，则调用nativeWake函数进行JNI方法调用发送消息。

##### 消息循环处理与回调
Looper中的loop()方法会拿到与looper耦合的MessageQueue对象，由MessageQueue调用next()方法，如果meesage对象不为空，则调用`msg.target.dispatchMessage(msg);`回调到Handler中。
`public void dispatchMessage(Message msg) {
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
    }`

其中，else语句中的handlerMessage最终会回调回当时创建的Handler对应的handMessage方法中。

*在回调中，我们注意到除了handleMessage的回调，还有可以注册message的CallBack或者Handler的callback来回调。message的Callback需要在message创建时传入一个Runnable对象。而Handler的Callback与handleMessage方法就目前而言使用上没感觉有什么区别。
希望有了解的同学能对这个问题留言说明下，谢谢～！*

##### 写到最后
>Handler的基础分析就算完成了，后续有时间会做其他相关的源码分析。
不足之处，请大家批评指导。

ps：Markdown代码框标记不怎么好用，大家有什么建议也可以留言(⊙o⊙)哦！
