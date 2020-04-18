# AsyncTask的版本变更之谜

都说孙悟空能七十二变，AsyncTask也可以说是Android届的孙悟空了，版本和特性经过多次改动，给很多人埋下了一些坑，下面简单介绍AsyncTask在不同版本上的表现。

## 1.6之前

串行执行，这个版本的设备已经灭绝，不过多介绍。

## 1.6-3.0之前

从1.6之后开始采用线程池来优化性能，线程池允许任务并行，即同时执行任务数大于1，这一点从代码中可以看出来，下面是2.3.5版本中的AsyncTask相关代码。

```text
private static final int CORE_POOL_SIZE = 5;
private static final int MAXIMUM_POOL_SIZE = 128;
private static final int KEEP_ALIVE = 1;
private static final BlockingQueue<Runnable> sWorkQueue =
            new LinkedBlockingQueue<Runnable>(10);

 private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,
            MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory);
```

从上面我们可以得出这样一个结论，在这个版本中，AsyncTask内部维护着一个线程池：

* 线程池的最小线程数为5
* 最大线程数为128
* 线程空闲存活时间为1s
* 线程池的缓存队列长度为10

那么这些参数设置在运行时会造成什么影响呢？

当通过`AsyncTask.execute()`添加任务到线程池时，会出现以下几种情况：

* 如果此时线程池中的数量小于corePoolSize，即使线程池中的线程都处于空闲状态，也会创建新的线程来处理被添加的任务
* 如果此时线程池中的数量等于corePoolSize，但是缓冲队列workQueue未满，那么任务被放入缓冲队列
* 如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue已满，并且线程池中的数量小于maximumPoolSize时，会创建新的线程来处理被添加的任务
* 如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue已满，并且线程池中的数量等于maximumPoolSize，那么就会调用默认的处理方式，在Android中默认处理类是AbortPolicy，将会抛出RejectedExecutionException
* 当线程池中的线程数量大于corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止，线程池负责动态的调整池中的线程数

由此我们可以看出，处理任务的优先级为：核心线程corePoolSize、任务队列workQueue、最大线程maximumPoolSize，如果三者都满了，使用handler处理被拒绝的任务。

由于在这个版本中是并发处理，并且存在线程池最大上限数，因此很容易出现一些问题。一个是老版本上串行处理的任务，如果存在执行顺序的话，并行处理就会带来一些逻辑上的问题，另外一个就是如果线程数量到达128的最大上限，那么就会Crash！这简直是一个噩梦！

## 3.0之后

在3.0及以后的版本上AsyncTask终于算是"人到中年"，慢慢稳定了下来。

在现在的版本中，AsyncTask的默认执行方式又换回了串行模式，同时也开放了自定义执行线程池的接口，避免了并行运行时Crash的悲剧。

不过在线程池的参数设置上面，新版本又进行了不小的改动。

```text
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE = 1;
private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

public static final Executor THREAD_POOL_EXECUTOR
        = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```

核心线程数变为了CPU核心数+1，最大线程数由128变为了CPU核心数\*2+1，而缓存队列由10增加到128。我猜测这样的改动是为了提高执行效率，根据CPU核心数来动态设置核心线程数更加合理，可以让每个CPU都能高效率的执行，同时限制了最大线程数，可以减少CPU在线程间频繁切换带来的性能损耗。

在这个版本中默认线程池是支持并发执行的，但是AsyncTask内部采取了一个策略实现了串行执行效果。

关于AsyncTask的使用和实现原理，我将在接下来的章节里面进行介绍，这里不再赘述。

