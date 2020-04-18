# Handler机制

当我面试别人的时候，通常问的第一个问题就是Handler机制，之所以会如此重视Handler机制，主要有以下三个原因：

1. Handler机制在Android开发中使用的非常频繁，也非常的重要，但使用不当会造成很多的问题，比如内存泄露
2. Android中的很多Framework模块都涉及到Handler机制，了解Handler机制就可以更加轻松的理解Android的其它功能模块
3. Handler机制并不算复杂，通过考察面试者对这个知识点的了解程度，可以了解面试者对待技术的态度和研究问题的深度，可以从一定程度上反应面试者的水平

这一章我将介绍Handler机制的实现原理、部分内部实现细节和Handler机制在Framework中的部分应用。

学习这一章之后，你应该对Android的消息处理机制有个完整的认识，并可以正确使用Handler机制。

注意，在这章的内容中，"Handler机制"指的是由Handler，Looper，Message，MessageQueue，ThreadLocal组成的整个消息处理机制，而"Handler"则指的是Handler.java这个具体类，请读者在阅读时注意区分。

Handler作为Handler机制对外的辅助类，通过观察Handler对外暴露的接口，我们可以了解到Handler机制的作用，总结来说，就是一句话“处理从任何其他线程发送过来的消息并进行处理”。

下面是Handler对外暴露的和消息发送有关的方法。

使用Handler.postXXX\(\)可以发送一个延时任务，从而在未来的某个时间点处理Message或者是执行Runnable任务。

主要的方法包括：

* post\(Runnable r\)
* postAtTime\(Runnable r, long uptimeMillis\)
* postAtTime\(Runnable r, Object token, long uptimeMillis\)
* postDelayed\(Runnable r, long delayMillis\)
* postAtFrontOfQueue\(Runnable r\)

这几个方法的实质是创建了一个Message对象，并且将传入的Runnable对象设置为Message.callback属性，从而等待在未来的某个时间点执行Runnable.run\(\)。

frameworks/base/core/java/android/os/Handler.java

```text
 private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

而使用sendXXX\(\)等方法则可以发送Message，从而在未来某个时间点收到Message完成后续操作。

主要的发送方法包括：

* sendMessage\(Message msg\)
* sendEmptyMessage\(int what\)
* sendEmptyMessageDelayed\(int what, long delayMillis\)
* sendEmptyMessageAtTime\(int what, long uptimeMillis\)
* sendMessageDelayed\(Message msg, long delayMillis\)
* sendMessageAtTime\(Message msg, long uptimeMillis\)
* sendMessageAtFrontOfQueue\(Message msg\)

通过postXXX\(\)与sendXXX\(\)都可以发送Message，区别在于Message.callback是否为空，而这个主要差别最终Handler.dispatchMessage\(Message\)中体现出来：

frameworks/base/core/java/android/os/Handler.java

```text
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

private static void handleCallback(Message message) {
        message.callback.run();
    }
```

因此，我们可以得出以下结论：

* 这两种方法的实质都是添加Message到MessageQueue中
* 使用Runnable只是因为Runnable接口中的run\(\)，并非开启新的线程
* handleMessage\(\)和Runnable.run\(\)的调用都是在Handler的绑定线程中，与当前所在线程无关

当Handler被实例化的时候，就与所在的线程紧密的联系在了一起，当在其他任意线程发送Message时，都可以在绑定线程中收到Message并进行分发，因此时常用来在工作线程中发送Message，在UI线程中完成界面的刷新。具体的实现机制将在后文中介绍。

