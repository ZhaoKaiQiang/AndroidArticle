# 使用Handler的正确姿势是什么？

在实际开发中，我们有时会像下面这样使用Handler。

```text
public class MainActivity extends Activity { 
    private final Handler mLeakyHandler = new Handler() { 
        @Override 
        public void handleMessage(Message msg) { 
            ... 
        } 
    } 
}
```

但是这样使用的时候，lint工具会提示警告

```text
In Android, Handler classes should be static or leaks might occur, Messages enqueued
on the application thread’s MessageQueue also retain their target Handler. 
If the Handler is an inner class, its outer class will be retained as well. 
To avoid leaking the outer class, declare the Handler as a static nested class 
with a WeakReference to its outer class
```

是的，这样使用Handler是有内存泄露的风险的。

因为Handler可以发送延迟Message，当Handler作为内部类被声明时，会隐式持有Activity的引用。如果当前Activity被finish，而Handler发送的Message未被全部处理时，由于Message的target属性指向Handler，Handler则引用Activity，所以在Message未被处理之前，Activity会因为被Handler隐式引用不能被GC正常回收，从而造成内存泄露。

不过这种泄露一般是暂时的，当Message被处理之后，会切断与Handler的引用关系，这时虽然Handler仍然具有Activity的引用，但是因为Handler属于Activity，Activity作为一个组件单元并不被外部所引用。当再次引发GC时，Activity不可达，GC就会将Activity及其内部所有的成员变量回收掉。

frameworks/base/core/java/android/os/Message.java

```text
void recycleUnchecked() {
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        //当Message被分发处理之后，会调用此方法将target置空，切断与Handler的联系
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

那么如何正确的使用Handler才能避免内存泄露的发生呢？答案就是静态类和弱引用。

```text
public class MainActivity extends Activity {

    private final SafeHandler mHandler = new SafeHandler(this);

    private static class SafeHandler extends Handler { 

        private final WeakReference<MainActivity> mActivity; 

        public SafeHandler(MainActivity activity) { 
            mActivity = new WeakReference<>(activity); 
        } 

        @Override 
        public void handleMessage(Message msg) { 
            MainActivity activity = mActivity.get(); 
            if (activity != null) { 
                ... 
            } 
        } 
    } 
}
```

当使用静态内部类时，不会引用Activity，而且由于使用WeakReference，所以当Activity被finish时，并不会阻止GC的回收，从而避免了内存泄露。

需要注意的是，由于`Activity.runOnUiThread()`不能发送延时Runnable，所以不存在内存泄露的场景，其Handler对象不需要设置为static。

framework/base/core/java/android/app/Activity.java

```text
final Handler mHandler = new Handler();
```

