# Message如何被分发到Handler.handlerMessage\(\)？

在Handler通过各种方法发送Message之后，最后都是将Message添加到了MessageQueue中。

frameworks/base/core/java/android/os/Handler.java

```text
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

下面简单分析一下MessageQueue.enqueueMessage\(\)的实现。

frameworks/base/core/java/android/os/MessageQueue.java

```text
 boolean enqueueMessage(Message msg, long when) {
        //可用性检查
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            //如果消息队列处于退出状态，则直接返回
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                msg.recycle();
                return false;
            }
            //标记Message为正在使用状态
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // mMessages为null时，将msg设置为head
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                //查找合适的位置，并将Message插入到队列中
                //合适的位置是指在队列的尾部或者是大于时间点Message的前面
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

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

至此，Message已经被插入到了MessageQueue的合适位置中。

在前面我们谈到，当`Looper.loop()`调用之后，Looper会调用`MessageQueue.next()`不断的从MessageQueue中取出Message交给Handler处理，下面我们接着分析一下这个过程。

frameworks/base/core/java/android/os/MessageQueue.java

```text
Message next() {

        int nextPollTimeoutMillis = 0;
        for (;;) {

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    //下一个Message还未准备好，设置唤醒时
                    if (now < msg.when) {
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 取得Message，则返回
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        return msg;
                    }
                } else {
                    nextPollTimeoutMillis = -1;
                }

                //正在退出状态，返回null
                if (mQuitting) {
                    dispose();
                    return null;
                }

            }

            nextPollTimeoutMillis = 0;
        }
    }
```

从上面的代码可以看出，当调用`MessageQueue.next()`的时候会阻塞住，只有等待Message.when符合条件的时候，才会返回。在这之后，`Looper.loop()`得以继续往下执行。

frameworks/base/core/java/android/os/Looper.java

```text
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        for (;;) {
            //可能阻塞住
            Message msg = queue.next();
            if (msg == null) {
                return;
            }
            //调用发送这个Message的Handler的dispatchMessage()
            msg.target.dispatchMessage(msg);
            //回复成员变量初始状态，并标记flags为FLAG_IN_USE
            msg.recycleUnchecked();
        }
    }
```

至此，一个Message从被发送到被处理的整个流程就很清晰了。

