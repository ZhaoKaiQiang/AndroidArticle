# Looper、Thread、ThreadLoacal的关系是什么？

前面有提到"Handler绑定的线程"这个概念，可能有点不好理解，因为在handler的源码中并没有和Thread相关的代码，完成Handler和Thread绑定的其实是Looper。

在前面我们得知，在Handler实例化的时候会通过`Looper.myLooper()`完成成员变量mLooper的初始化操作。

frameworks/base/core/java/android/os/Handler.java

```text
    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
```

从注释也可以看出，这个方法是返回和当前线程关联的Looper对象，如果没有Looper与当前Thread关联，就会返回null，而在Handler的构造函数中，就会因为Looper为null而抛出一个很常见的异常。

frameworks/base/core/java/android/os/Handler.java

```text
mLooper = Looper.myLooper();
if (mLooper == null) {
    throw new RuntimeException(
        "Can't create handler inside thread that has not called Looper.prepare()");
}
```

这也就是说，如果我们想使用Handler，那么就要保证当前线程必须有相关联的Looper。

那么为什么在UI线程使用的时候，没有关联Looper也可以正常使用Handler呢？

这是因为在UI线程开启的时候，就已经关联了Looper。

frameworks/base/core/java/android/app/ActivityThread.java

```text
 public static void main(String[] args) {

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

在程序退出之前会一直阻塞在`Looper.loop()`，如果主线程的消息轮询退出会引发RuntimeException。

在`Looper.prepareMainLooper()`的执行过程中会调用`Looper.prepare()`完成sMainLooper的初始化,并且保证只初始化一次。

frameworks/base/core/java/android/os/Looper.java

```text
public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

需要注意的是，sMainLooper是一个静态变量，因此整个程序中只会存在一个实例，我们可以通过下面的方法在任意地方获取到Main Looper。由于存在多线程访问的场景，所以需要进行同步操作。

frameworks/base/core/java/android/os/Looper.java

```text
 private static Looper sMainLooper; 

 public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
```

因为sMainLooper与主线程相关联，因此可以定义一个Handler，很轻松的完成从任意线程到UI线程的切换。

```text
public final class HandlerFactory {

    private final static Handler INSTANCE = new Handler(Looper.getMainLooper());

    private HandlerFactory() {}

    public static Handler getInstance() {
        return INSTANCE;
    }
}
```

通过`HandlerFactory.getInstance().post(Runnable)`就可以保证Runnable运行在UI线程。

不管是哪种方式，最终都是通过`Looper.prepare()`完成了Looper的初始化和线程的关联操作。

frameworks/base/core/java/android/os/Looper.java

```text
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

那么，为什么通过`Looper.myLooper()`就能获取到和当前Thread关联的Looper对象呢？

在`Looper.prepare()`中，会通过`sThreadLocal.set(new Looper(quitAllowed))`来保存Looper，之后所有关于Looper的获取都是通过`sThreadLocal.get()`完成的，所以ThreadLocal应该很重要。

下面看下在`ThreadLocal.set()`和`ThreadLocal.get()`分别发生了什么。

java/lang/ThreadLocal.java

```text
 public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

在`set()`中，会从当前Thread中取出ThreadLocalMap对象，这个对象默认是null，Thread并不对这个成员变量进行维护，全部的操作只由ThreadLocal完成。因此之后会调用`createMap()`，并将Thread和valus进行处理后存入，这样就完成了Thread和Looper的映射关系。在`get()`中的操作会把Thread作为key，找出对应的value，这也就是为什么通过`Looper.myLooper()`可以在任意线程获取到对应Looper的原因。

最后一句话总结Looper、Thread、ThreadLocal的关系：开启消息循环的Thread必须绑定一个Looper，Looper与Thread的映射关系存储在ThreadLocal中。

