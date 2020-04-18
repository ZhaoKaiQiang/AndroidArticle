# Message是如何实现复用的？

为了提高效率，我们通常可以通过`Handler.obtainMessage()`来获取一个Message，实际是通过下面获取的。

frameworks/base/core/java/android/os/Handler.java

```text
public final Message obtainMessage(){
    return Message.obtain(this);
}
```

frameworks/base/core/java/android/os/Message.java

```text
public static Message obtain(Handler h) {
        Message m = obtain();
        m.target = h;
        return m;
    }

public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

Message除了可以作为发送信息的载体之外，其内部还维持了一个最大容量为50的单链表结构作为缓存池。

frameworks/base/core/java/android/os/Message.java

```text
    private static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;
    private static final int MAX_POOL_SIZE = 50;
```

因为这些都是静态变量，所以Message缓存池的概念在整个程序里面只有一个。

由于`Message.obtian()`的调用存在多个线程之间同时访问静态变量的数据共享问题，所以需要对方法块进行同步操作。从上面的代码中可以很清楚的看到对单链表的操作，如果存在缓存Message的话，就取出Header Message，并将Header指向Header.next。

那么sPool是在什么时候添加进Message的呢？

在Looper.loop\(\)中，如果Message被处理之后，会调用下面的方法，完成数据的初始化和添加进缓存池的操作：

frameworks/base/core/java/android/os/Looper.java

```text
 public static void loop() {
    ...
    msg.target.dispatchMessage(msg);
    msg.recycleUnchecked();
 }
```

frameworks./base/core/java/android/os/Message.java

```text
 void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

因此，通过这种机制就完成了Message的缓存池功能，提高了效率。

