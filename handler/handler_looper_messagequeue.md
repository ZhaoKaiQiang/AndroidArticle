# Handler、Looper、MessageQueue的关系是什么？

## 类之间的关系

在Handler中持有MessageQueue和Looper的成员变量。

frameworks/base/core/java/android/os/Handler.java

```text
 final MessageQueue mQueue;
 final Looper mLooper;
```

在Handler的构造函数被调用时，完成了这两个成员变量的初始化操作。

frameworks/base/core/java/android/os/Handler.java

```text
public Handler(Callback callback, boolean async) {
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
    }
```

从源码可以看出，Handler中的成员变量mQueue和mLooper中的成员变量mQueue都指向同一个对象。

## 功能之间的关系

Handler、Looper、MessageQueue这三个主要的类构成了Android中的Handler机制，它们的主要作用分别是：

* Handler负责发送、处理Message
* MessageQueue负责维护Message队列
* Looper负责从MessageQueue中不断地取出Message，并分发给Handler去处理

