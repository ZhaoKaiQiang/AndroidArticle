# HandlerThread和IntentService

为什么要把HandlerThread和IntentService放在一起说呢？因为这两个类的关系紧密。

## HandlerThread到底是什么？

一句话就是：有消息循环的Thread就是HandlerThread。

HandlerThread的作用就是不断的从消息队列中取出Message，然后在当前线程执行。

## HandlerThread的实现原理

其实HandlerThread的源代码非常简单，只有150行左右。

当我们在使用HandlerThread的时候，我们一般需要按照下面的步骤：

```text
private Handler mHandler ;
private Handler.Callback mCallback = new Handler.Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        return false;
    }
};

public void  initHanlderThread(){
    HandlerThread workHandle = new HandlerThread("workHandleThread");
    workHandle.start();
    mHandler = new Handler(workHandle.getLooper(), mCallback);
}
```

之后使用`mHandler.postXXX()`就可以将任务发送至HandlerThread执行了。

HandlerThread继承自Thread，当执行start\(\)的时候，run\(\)就会调用，在这里实现了消息队列的创建：

```text
public class HandlerThread extends Thread {

    Looper mLooper;

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }

     public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }

        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
}
```

可以看到，在run\(\)里面完成了Looper消息队列的创建，所以在实例化Handler的时候，Handler就会与HandlerThread线程的Looper进行关联，由Handler发送的Message最终会在Looper里面进行分发和执行。

同时，为了保证Looper一定可以初始化，这里使用wait\(\)和notifyAll\(\)进行了线程同步操作，需要注意的是，wait\(\)阻塞的并不是HandlerThread的线程，而是调用getLooper\(\)所在的线程，在这里指的就是UI线程，虽然有可能阻塞UI线程，但是正常情况下这里的操作没有什么危险。

通过简单的查看HandlerThread源码，我们可以很清楚HandlerThread的实现原理。

## 在什么时候用HandlerThread？

如果要想知道什么时候应该用HandlerThread，那么我们应该先总结一下HandlerThread的特点：

* 是一个线程
* 拥有Handler机制下消息循环
* 由于基于是Handler机制，所以所有的消息处理都是串行的

有了这些特点，我们就不难知道HandlerThread在什么情况下使用比较合适了。

* UI线程实际上就是一个HandlerThread，所有View的操作都是在消息循环里面处理的，如果在UI线程里面做耗时操作，比如大文件读取，就会阻塞消息队列，造成ANR，这也是为什么我们不能在UI线程中进行耗时工作的最根本因
* 可以使用HandlerThread来模拟『单线程+队列』结构，HandlerThread就相当于是一个SingleThreadExecutor结构，即单线程池，在App启动的时候，如果有很多初始化工作不需要在UI线程完成，那么可以将这些任务发送至HandlerThread，然后顺序完成，加快应用的启动速度

在前面分析Picasso的时候，我们就看到分发器Dispatcher里面有DispatcherThread来完成任务的分发，这样对任务的处理就转换到了另外一个线程，从而不会阻塞UI线程。

## HandlerThread与IntentService有什么关系？

简单一句话：IntentService的核心功能是由HandlerThread实现的。

IntentService的特点就是：任务完成后自动停止，而且不需要手动开启新的线程，默认是在工作线程完成。

如果要想知道IntentService是如何完成这个功能的，我们还是需要去翻看源码,不过IntentService也很简单，只有160行左右。

在查看源码之前，简单回顾一下IntentService的用法。

```text
public class DownloadServices extends IntentService {

    @Override
    protected void onHandleIntent(Intent intent) {

    }

}
```

IntentService和普通的Service使用方法完全一样，使用`ContextWrapper.startService()`即可启动IntentService，因为IntentService是抽象类，因此需要直接继承，在onHandlerIntent\(\)中会可以接收到Intent信息，需要注意的是，onHandlerIntent\(\)是运行在工作线程的。

下面我们从源码的角度来学习IntentService的实现原理。

```text
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    protected abstract void onHandleIntent(Intent intent);

}
```

因为IntentService继承自Service，因此Service的特性在这里也是完全通用的，这主要表现在Service的生命周期上。

当Service启动时，会先调用onCreate\(\)完成初始化操作，在这里则是完成了HandlerThread和Handler的初始化及绑定操作，之后会调用onStartCommand\(\)，间接在onStart\(\)中使用mServiceHandler来发送Message，在\`\`\`Handler.handleMessage\(\)中会调用onHandleIntent\(\)完成任务的执行。

由于内部实现使用的HandlerThread，因此HandlerThread的特性在这里仍然适用，具体来说就是指任务是以**单线程串行**的方式处理的。

在源码中我们可以发现，在onHandleIntent\(\)执行之后，就会调用stopSelf\(msg.arg1\)来停止Service，这也是为什么IntentService不需要手动stop的原因。

还有一个问题值得我们关注，那就是如果在onHandleIntent\(\)执行过程中，又发起了一次请求的话，IntentService是如何处理的呢？下面是这种情况的模拟，为了模拟耗时情况，在onHandleIntent\(\)中手动延迟了3s。

```text
@Override
    protected void onHandleIntent(Intent intent) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Log.d(TAG, "onHandleIntent");
    }
```

下面是测试结果：

```text
D/IntentService: constructor
D/IntentService: onCreate
D/IntentService: onStartCommand
D/IntentService: onStart
D/IntentService: onStartCommand
D/IntentService: onStart
D/IntentService: onStartCommand
D/IntentService: onStart
D/IntentService: onHandleIntent
D/IntentService: onHandleIntent
D/IntentService: onHandleIntent
D/IntentService: onDestroy
```

因此我们可以得出结论：当onHandleIntent\(\)未结束情况下再次发起请求时，会调用onStartComman\(\)将任务缓存在消息队列，等待所有的任务执行完毕后，才会stop\(\)结束IntentService。

至此我们对HandlerThread和IntentService的实现原理及用法应该有一个比较全面的认识了。

