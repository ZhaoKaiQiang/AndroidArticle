# Picasso是如何实现异步加载图片的？

图片加载的需求在Android中是非常常见的需求，同时由于Bitmap是内存大户，所以如何高效的加载图片是一个非常重要的问题。目前来说，UIL、Glide、Picasso、Fresco等优秀的图片加载框架都可以满足我们的需求。本文以开源图片加载库Picasso为例，介绍图片加载库的通用设计框架及部分细节处理。

## Picasso简单使用

在介绍整体架构之前，我们先看一下使用Picasso如何完成一张图片的加载。

最简单情况下，使用一句代码即可完成一张图图片的加载过程，包括发起网络请求、处理返回值、对文件进行本地硬盘缓存和内存缓存、维护线程池等操纵。

```text
Picasso.with(context).load(URL).into(imageView);
```

下面我讲从源代码的角度，讲解这一行代码的底层到底做了哪些操作，以便于我们对通用图片加载框架有比较完整的了解。

## Picasso的初始化操作

Picasso是面向开发者的一个接口，这个接口的意思并非是`Interface`，而是指如果想完成图片加载工作，我们就需要和Picasso打交道。你还记得前面我说过的Activity吗？Activity也是一个面向开发者的接口，我们用Activity来完成界面的显示、交互、销毁功能，这里我们就用Picasso完成图片加载的功能，是一样的思想。

好了，我们要从Picasso开始入手了！

当调用`Picasso.with(context)`的时候，系统会完成Picasso对象的实例化操作。

```text
static volatile Picasso singleton = null;

public static Picasso with(Context context) {
    if (singleton == null) {
      synchronized (Picasso.class) {
        if (singleton == null) {
          singleton = new Builder(context).build();
        }
      }
    }
    return singleton;
  }
```

在这里，使用了线程安全的单例模式完成了Picasso的初始化，同时，由于Picasso的初始化过程可以自定义很多参数，所以使用了建造者模式完成Picasso对象的构建。

使用这种方式会调用默认的参数来构造:

```text
public Picasso build() {
      Context context = this.context;

      if (downloader == null) {
        downloader = Utils.createDefaultDownloader(context);
      }
      if (cache == null) {
        cache = new LruCache(context);
      }
      if (service == null) {
        service = new PicassoExecutorService();
      }
      if (transformer == null) {
        transformer = RequestTransformer.IDENTITY;
      }

      Stats stats = new Stats(cache);

      Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);

      return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
          defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
    }
```

下载器downloader的设置策略为：若当前项目中依赖okhttp，则使用`OkHttpDownloader`，否则使用`HttpURLConnection`实现的`UrlConnectionDownloader`，这一点从`Utils.createDefaultDownloader(context)`的实现过程可以看出来。

```text
static Downloader createDefaultDownloader(Context context) {
    try {
      Class.forName("com.squareup.okhttp.OkHttpClient");
      return OkHttpLoaderCreator.create(context);
    } catch (ClassNotFoundException ignored) {
    }
    return new UrlConnectionDownloader(context);
  }
```

内存缓存cache的默认实现是LruCache，关于LruCache的具体实现将在**内存管理机制**一章中详细介绍。

线程池service的默认实现为`PicassoExecutorService`，它继承自ThreadPoolExecutor，主要添加了根据当前设备的网络状况调整**线程池核心线程数**的功能。关于这一功能点的实现机制我将在下面章节中介绍。

请求转换器transformer的默认实现并未对Request进行任何处理，如果有需求的话，我么可以自定义转换器，然后对Request统一进行处理，比如对请求连接地址进行统一处理等等。

```text
public interface RequestTransformer {

    Request transformRequest(Request request);
    //请求转换器的默认实现并未做任何操作
    RequestTransformer IDENTITY = new RequestTransformer() {
      @Override public Request transformRequest(Request request) {
        return request;
      }
    };
  }
```

除此之外，Stats负责统计信息，如cache命中率、丢失率、下载文件总大小、总数量等参数，任务分发器Dispatcher负责任务的分发等工作，这几个类的具体功能和实现将在下面介绍。

至此，Picasso单例的初始化动作算是完成了，下面可以使用中Picasso发起请求，完成图片的加载工作了。

## 创建图片加载请求

前文说到，使用`Picasso.with(context)`完成了单例对象的创建，接下来就应该使用`Picasso.load(URL)`发起图片加载请求了。

Picasso

```text
public RequestCreator load(String path) {
    if (path == null) {
      return new RequestCreator(this, null, 0);
    }
    if (path.trim().length() == 0) {
      throw new IllegalArgumentException("Path must not be empty.");
    }
    return load(Uri.parse(path));
  }

public RequestCreator load(Uri uri) {
    return new RequestCreator(this, uri, 0);
}
```

通过`Picasso.load()`可以创建RequestCreator对象，RequestCreator负责完成请求的发起工作。这里需要注意的一点是，请求地址Path可以为null，这个时候不会发起任何请求，如果设置占位图片的话，会显示占位图片，但是Path的内容不能为空，这会引发Crash，这一点需要特别注意，调用之前需要对Path进行判断以免引发Crash。

可以通过以下方式设置占位图片

```text
Picasso.load(URL).placeholder(R.mipmap.ic_launcher);
```

通过`Picasso.load(URL)`完成RequestCreator的创建，在构造函数中会完成以下操作：

```text
private final Picasso picasso;
private final Request.Builder data;

RequestCreator(Picasso picasso, Uri uri, int resourceId) {
    if (picasso.shutdown) {
      throw new IllegalStateException(
          "Picasso instance already shut down. Cannot submit new requests.");
    }
    this.picasso = picasso;
    this.data = new Request.Builder(uri, resourceId, picasso.defaultBitmapConfig);
  }
```

在这个过程中，又完成了`Request.Builder`实例的创建，`Request.Builder`用于接受Request的自定义参数，完成Request的创建。

这里需要明确的一点是，每次发起一个图片请求，都会创建一个RequestCreator。

实际上，当获取到RequestCreator对象之后，我们可以通过链式调用自定义很多的参数，包括设置tag，设置图片对齐模式，设置加载出错图片，设置占位图片，旋转，自定义大小，图片自定义处理等操作。

![](http://i4.tietuku.com/2bf283260495ebf3.png)

那么怎么构造这么复杂的对象呢？是的，建造者模式。实际上，在设置这些参数的时候，改变的都是`Request.Builder`对象，这样在创建Request的时候，就可以将这些自定义参数传递过去。

为了实现链式调用，在设置完参数之后，都将当前对象作为返回值返回。

```text
public RequestCreator centerCrop() {
    data.centerCrop();
    return this;
  }

  public RequestCreator onlyScaleDown() {
    data.onlyScaleDown();
    return this;
  }

  public RequestCreator rotate(float degrees) {
    data.rotate(degrees);
    return this;
  }
```

Ok，现在我们已经把所有的参数都设置好了，接下来该做什么了？是的，创建请求，然后将请求发送出去。

使用RequestCreator发送图片请求有两种方式，这两种方式用于不同场景。

* RequestCreator.into\(target\)，通过这种方式可以直接完成图片在target上的显示，最常用的也是这种用法
* RequestCreator.fetch\(\)，通过这种方式可以完成图片的预加载工作，即将目标图片下载至本地，并加入到内存缓存中，这样当下次使用上面的方式加载相同图片时，会直接从内存缓存中获取，极大的提高加载速度

下面将介绍使用第一种方式发送Request的流程。

```text
//使用这种方式会持有target的若引用，并且支持对象回收
public void into(ImageView target) {
    into(target, null);
  }

  //如果使用这种方式，Callback会持有Activity或者是Fragment的强引用
  //因此我们需要在合适的位置调用Picasso.cancelRequest()取消任务，防止内存泄露
  public void into(ImageView target, Callback callback) {
    long started = System.nanoTime();
    //检查当前线程是否是主线程
    checkMain();

    if (target == null) {
      throw new IllegalArgumentException("Target must not be null.");
    }

    //如果没有设置url地址或者是本地资源id，则取消加载任务，直接显示占位图片
    //这解释了为何前面说url地址可以为null
    if (!data.hasImage()) {
      picasso.cancelRequest(target);
      if (setPlaceholder) {
        setPlaceholder(target, getPlaceholderDrawable());
      }
      return;
    }

    //deferred代表是否延迟加载
    //当使用fit()来设置Bitmap与ImageView大小相同时，条件满足
    if (deferred) {
      //如果想Bitmap与ImageView大小相同，则不能通过resize()再次设置Bitmap的大小
      if (data.hasSize()) {
        throw new IllegalStateException("Fit cannot be used with resize.");
      }
      int width = target.getWidth();
      int height = target.getHeight();
      //这种情况下，ImageView还未初始化完成
      //所以先设置占位图，并发送延迟任务
      if (width == 0 || height == 0) {
        if (setPlaceholder) {
          setPlaceholder(target, getPlaceholderDrawable());
        }
        picasso.defer(target, new DeferredRequestCreator(this, target, callback));
        return;
      }
      //ImageView已初始化完毕，直接设置大小
      data.resize(width, height);
    }
    //这里使用Builder和Transformer来创建Request
    Request request = createRequest(started);
    //根据request拼接了好多参数作为Key
    String requestKey = createKey(request);
    //是否从内存缓存中读取
    if (shouldReadFromMemoryCache(memoryPolicy)) {
      Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);
      if (bitmap != null) {
        //取消请求，直接设置
        picasso.cancelRequest(target);
        setBitmap(target, picasso.context, bitmap, MEMORY, noFade, picasso.indicatorsEnabled);
        if (picasso.loggingEnabled) {
          log(OWNER_MAIN, VERB_COMPLETED, request.plainId(), "from " + MEMORY);
        }
        if (callback != null) {
          callback.onSuccess();
        }
        return;
      }
    }
    //默认setPlaceholder=true，因此会加载占位图
    if (setPlaceholder) {
      setPlaceholder(target, getPlaceholderDrawable());
    }
    //生成真正的请求
    Action action =
        new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
            errorDrawable, requestKey, tag, callback, noFade);
    //将请求添加到队列中
    picasso.enqueueAndSubmit(action);
  }
```

这里需要重点看的是，Request的生成过程。

```text
private Request createRequest(long started) {
    //每一个请求都有一个唯一id，且线程安全
    int id = nextId.getAndIncrement();
    //利用Builder中的数据创建Request
    Request request = data.build();
    request.id = id;
    request.started = started;

    ...
    //这里就用了Picasso的请求转换器，默认未作任何处理
    Request transformed = picasso.transformRequest(request);
    if (transformed != request) {
      transformed.id = id;
      transformed.started = started;
    }

    return transformed;
  }
```

至此，一个图片请求被封装成了ImageViewAction，添加到了Picasso的请求队列中。

## Picasso如何完成请求的分发

在上文中，通过`Picasso.enqueueAndSubmit()`完成了ImageViewAction到线程池的添加：

```text
final Map<Object, Action> targetToAction;
final Map<ImageView, DeferredRequestCreator> targetToDeferredRequestCreator;

 void enqueueAndSubmit(Action action) {
    Object target = action.getTarget();
    if (target != null && targetToAction.get(target) != action) {
      cancelExistingRequest(target);
      targetToAction.put(target, action);
    }
    submit(action);
  }

   private void cancelExistingRequest(Object target) {
    checkMain();
    Action action = targetToAction.remove(target);
    if (action != null) {
      action.cancel();
      dispatcher.dispatchCancel(action);
    }
    if (target instanceof ImageView) {
      ImageView targetImageView = (ImageView) target;
      DeferredRequestCreator deferredRequestCreator =
          targetToDeferredRequestCreator.remove(targetImageView);
      if (deferredRequestCreator != null) {
        deferredRequestCreator.cancel();
      }
    }
  }
```

在`Picasso.enqueueAndSubmit()`中，会首先对请求进行检测，如果有相同的请求的话，就会取消已经存在的请求，并把新请求添加到targetToAction中，targetToAction保存的是target与Action之间的映射，根据target来区分是否是同一个请求。如果target是ImageView类型，还要从targetToDeferredRequestCreator中去除对应数据，targetToDeferredRequestCreator保存的是延时任务，即设置fit\(\)属性的任务。

将这一切的准备工作完成之后，就利用`Picasso.submit(action)`提交任务了。

```text
 final Dispatcher dispatcher;

void submit(Action action) {
    dispatcher.dispatchSubmit(action);
  }
```

还记得前面说过的Diapatcher的作用吗？Diapatcher是用来完成任务分发的，下面重点看一下任务分发的流程。

首先Diapatch是Picasso的成员变量，由于Picasso是单例模式，因此dispatch也只会存在一个实例，默认的dispatch是在Picasso单例初始化的时候实例化的，而且Picasso不支持Diapatch的自定义，只能使用默认实现。

```text
public Picasso build() {
     Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);
 }
```

从Dispatcher构造函数的参数来看，Diapatcher的功能应该是很强大的，因为与很重要的几个模块都扯上了关系，比如线程池，Handler，网络下载器，内存缓存，状态统计等。

Dispatcher在构造时确实完成了很多的工作。

```text
  final DispatcherThread dispatcherThread;
  final Context context;
  final ExecutorService service;
  final Downloader downloader;
  final Map<String, BitmapHunter> hunterMap;
  final Map<Object, Action> failedActions;
  final Map<Object, Action> pausedActions;
  final Set<Object> pausedTags;
  final Handler handler;
  final Handler mainThreadHandler;
  final Cache cache;
  final Stats stats;
  final List<BitmapHunter> batch;
  final NetworkBroadcastReceiver receiver;
  final boolean scansNetworkChanges;

  boolean airplaneMode;

  Dispatcher(Context context, ExecutorService service, Handler mainThreadHandler,
      Downloader downloader, Cache cache, Stats stats) {
      //本质是一个HandlerThread
    this.dispatcherThread = new DispatcherThread();
    this.dispatcherThread.start();
    Utils.flushStackLocalLeaks(dispatcherThread.getLooper());
    this.context = context;
    this.service = service;
    this.hunterMap = new LinkedHashMap<String, BitmapHunter>();
    this.failedActions = new WeakHashMap<Object, Action>();
    this.pausedActions = new WeakHashMap<Object, Action>();
    this.pausedTags = new HashSet<Object>();
    this.handler = new DispatcherHandler(dispatcherThread.getLooper(), this);
    this.downloader = downloader;
    this.mainThreadHandler = mainThreadHandler;
    this.cache = cache;
    this.stats = stats;
    this.batch = new ArrayList<BitmapHunter>(4);
    //是否是飞行模式
    this.airplaneMode = Utils.isAirplaneModeOn(this.context);
    //检测网络权限
    this.scansNetworkChanges = hasPermission(context, Manifest.permission.ACCESS_NETWORK_STATE);
    //用于根据网络动态动态的修改核心线程数
    this.receiver = new NetworkBroadcastReceiver(this);
    receiver.register();
  }
```

Ok，这里可能有些成员变量不太好理解作用，不过没关系，在下面的流程中将会重点介绍。我们看一下`dispatcher.dispatchSubmit(action)`之后的操作。

```text
void dispatchSubmit(Action action) {
    handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
  }
```

有一个问题值得我们思考，这里为什么要用Handler呢？

要回答这个问题，我们需要先弄明白：handler是与哪个线程绑定的？

从代码中可以很容易得到答案，handler是和dispatcherThread绑定的，消息会在dispatcherThread进行处理。

```text
this.handler = new DispatcherHandler(dispatcherThread.getLooper(), this);
```

dispatcherThread是什么？就是一个简单的HandlerThread：

```text
static class DispatcherThread extends HandlerThread {
    DispatcherThread() {
      super(Utils.THREAD_PREFIX + DISPATCHER_THREAD_NAME, THREAD_PRIORITY_BACKGROUND);
    }
  }
```

那么为什么这么做呢？我觉得有两个作用：

1. 将消息从主线程发送至工作线程，可以不影响主线程消息队列的处理速度
2. Diapatcher的工作不需要和UI打交道，这样可以减缓主线程的压力

通过Handler从主线程发送消息至工作线程，这也是Handler的一个重要用法，需要重点掌握。

Ok，现在下面的工作都是在工作线程DispatcherThread中执行了。

```text
private static class DispatcherHandler extends Handler {
    private final Dispatcher dispatcher;

    public DispatcherHandler(Looper looper, Dispatcher dispatcher) {
      super(looper);
      this.dispatcher = dispatcher;
    }

    @Override public void handleMessage(final Message msg) {
      switch (msg.what) {
        case REQUEST_SUBMIT: {
          Action action = (Action) msg.obj;
          dispatcher.performSubmit(action);
          break;

          ...

        }
}

void performSubmit(Action action) {
    performSubmit(action, true);
  }

  void performSubmit(Action action, boolean dismissFailed) {
     //如果tag在暂停列表中，则将Action缓存起来，直接返回
    if (pausedTags.contains(action.getTag())) {
       //pausedActions的实现结构是WeakHashMap，因此并不会阻止ImageView的回收
       //这样可以防止内存泄露
      pausedActions.put(action.getTarget(), action);
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_PAUSED, action.request.logId(),
            "because tag '" + action.getTag() + "' is paused");
      }
      return;
    }

    BitmapHunter hunter = hunterMap.get(action.getKey());
    if (hunter != null) {
      //如果之前有添加过相同任务，绑定后直接返回
      hunter.attach(action);
      return;
    }

    if (service.isShutdown()) {
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_IGNORED, action.request.logId(), "because shut down");
      }
      return;
    }
    //根据不同的处理器创建对应的BitmapHunter
    hunter = forRequest(action.getPicasso(), this, cache, stats, action);
    //提交到线程池
    hunter.future = service.submit(hunter);
    //将Action进行缓存，加速下次的获取
    hunterMap.put(action.getKey(), hunter);
    if (dismissFailed) {
      //在失败集合中删除对应记录
      failedActions.remove(action.getTarget());
    }

    if (action.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_ENQUEUED, action.request.logId());
    }
  }
```

从上面的流程中，我们可以很清楚的的看到一个任务是如何被添加到线程池的，但是这里还有两个疑问：

1. BitmapHunter是什么？有什么作用？
2. 任务是如何被分发到合适的处理器的？由谁来决定从哪个渠道进行加载？比如资源、本地硬盘、Asses文件夹、网络等。

下面我来回答这两个问题。

首先BitmapHunter实现了Runnable，当他被扔进线程池之后，就会有线程来找他，并运行run\(\)。BitmapHunter的主要作用就是从对应渠道获取到Bitmap。

```text
class BitmapHunter implements Runnable {
}
```

至于任务是如何被分发给合适的处理器，这个过程也比较简单，在获取BitmapHunter对象的时候，是通过`BitmapHunter.forRequest()`来获取的，下面看下代码实现。

BitmapHunter\#forRequest

```text
static BitmapHunter forRequest(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats,
      Action action) {
    Request request = action.getRequest();
    List<RequestHandler> requestHandlers = picasso.getRequestHandlers();

    for (int i = 0, count = requestHandlers.size(); i < count; i++) {
      RequestHandler requestHandler = requestHandlers.get(i);
      if (requestHandler.canHandleRequest(request)) {
        return new BitmapHunter(picasso, dispatcher, cache, stats, action, requestHandler);
      }
    }

    return new BitmapHunter(picasso, dispatcher, cache, stats, action, ERRORING_HANDLER);
  }
```

通过遍历RequestHandler集合来创建合适的BitmapHunter，既然是遍历，那就肯定有先后顺序，这个是靠Picasso初始化的时候确定的。

```text
 private final List<RequestHandler> requestHandlers;


  Picasso(Context context, Dispatcher dispatcher, Cache cache, Listener listener,
      RequestTransformer requestTransformer, List<RequestHandler> extraRequestHandlers, Stats stats,
      Bitmap.Config defaultBitmapConfig, boolean indicatorsEnabled, boolean loggingEnabled) {

     //内置的默认处理器有7种
    int builtInHandlers = 7; 
    //我们可以添加自定义的RequestHandler
    int extraCount = (extraRequestHandlers != null ? extraRequestHandlers.size() : 0);
    //根据处理器的数量初始化固定长度集合，这样可以达到效率的最大化和内存占用最小化
    //同时注意选用的数据结构为ArrayList，保证了添加进去的数据有明确的顺序，这决定了每一个处理器的优先级
    //同时注意在遍历requestHandlers的过程中使用的是for循环而不是Iterator，对于Arraylist这种数据结构，可以提高遍历速度
    List<RequestHandler> allRequestHandlers =
        new ArrayList<RequestHandler>(builtInHandlers + extraCount);
    //应用内资源的处理优先级最高，这样可以让后面的处理器只需要对uri进行非空判断，因为resourse id 一定没有设置
    allRequestHandlers.add(new ResourceRequestHandler(context));
    if (extraRequestHandlers != null) {
      allRequestHandlers.addAll(extraRequestHandlers);
    }
    allRequestHandlers.add(new ContactsPhotoRequestHandler(context));
    allRequestHandlers.add(new MediaStoreRequestHandler(context));
    allRequestHandlers.add(new ContentStreamRequestHandler(context));
    allRequestHandlers.add(new AssetRequestHandler(context));
    allRequestHandlers.add(new FileRequestHandler(context));
    //只有所有本地数据都不能进行处理时，才会网络获取，网络获取的优先级最低
    allRequestHandlers.add(new NetworkRequestHandler(dispatcher.downloader, stats));
    //通过这个操作保证了requestHandlers不可再次被改变，否则会抛出异常
    requestHandlers = Collections.unmodifiableList(allRequestHandlers);
  }
```

从上面这些代码我们可以获取到非常多的信息，也可以看出Picasso在实现细节上处理上也非常优雅，从数据结构的选择、便利方式的选择、集合最大长度的设定、集合不可变的处理上都非常值得我们学习。

同时，在上面的代码里面我们可以看出，Picasso支持非常多途径的图片加载，包括：

* 系统内资源
* 联系人头像
* 媒体库
* ContentResolver
* Asset
* 本地文件
* 网络
* 自定义渠道

那么如何判定一个请求应该由谁进行处理呢？所有的处理器都继承子RequestHandler，并且重写了下面的方法

```text
public abstract boolean canHandleRequest(Request data);
```

因此我们只需要根据每种处理器的方法实现即可，不可能列出所有的代码，我只给出个别的实现，其他类似。

NetworkRequestHandler负责处理网络请求，所以它对uri的schem进行了处理：

```text
class NetworkRequestHandler extends RequestHandler {

     private static final String SCHEME_HTTP = "http";
       private static final String SCHEME_HTTPS = "https";

    @Override 
    public boolean canHandleRequest(Request data) {
            String scheme = data.uri.getScheme();
            return (SCHEME_HTTP.equals(scheme) || SCHEME_HTTPS.equals(scheme));
      }
}
```

联系人头像使用的是ContentProvider获取，所以处理方式如下：

```text
class ContactsPhotoRequestHandler extends RequestHandler {

    private static final UriMatcher matcher;

      static {
        matcher = new UriMatcher(UriMatcher.NO_MATCH);
        matcher.addURI(ContactsContract.AUTHORITY, "contacts/lookup/*/#", ID_LOOKUP);
        matcher.addURI(ContactsContract.AUTHORITY, "contacts/lookup/*", ID_LOOKUP);
        matcher.addURI(ContactsContract.AUTHORITY, "contacts/#/photo", ID_THUMBNAIL);
        matcher.addURI(ContactsContract.AUTHORITY, "contacts/#", ID_CONTACT);
        matcher.addURI(ContactsContract.AUTHORITY, "display_photo/#", ID_DISPLAY_PHOTO);
      }

      @Override 
      public boolean canHandleRequest(Request data) {
        final Uri uri = data.uri;
        return (SCHEME_CONTENT.equals(uri.getScheme())
        && ContactsContract.Contacts.CONTENT_URI.getHost().equals(uri.getHost())
        && matcher.match(data.uri) != UriMatcher.NO_MATCH);
  }
}
```

经过以上操作，一个包含有合适的RequestHandler的BitmapHunter就创建完毕了，接下来，就通过`service.submit(hunter)`被无情的扔进了线程池！

## 线程池的那些猫腻

Picasso的线程池在初始化的时候就设定好了，而且全局只有一个线程池，实现类是PicassoExecutorService，下面说一下Picasso在线程池上的处理。

PicassoExecutorService的处理非常简单，默认的核心线程数为3，最大线程数也为3，这也就决定了如果发起的请求多余3个的话，剩下的请求就会被缓存在PriorityBlockingQueue里面。

```text
class PicassoExecutorService extends ThreadPoolExecutor {
  private static final int DEFAULT_THREAD_COUNT = 3;

  PicassoExecutorService() {
    super(DEFAULT_THREAD_COUNT, DEFAULT_THREAD_COUNT, 0, TimeUnit.MILLISECONDS,
        new PriorityBlockingQueue<Runnable>(), new Utils.PicassoThreadFactory());
  }
}
```

关于核心线程数的处理上有一点值得思考，前面说过，AsyncTask也是有一个全局的线程池，在核心线程数的选择上是根据CPU个数动态变化的，个人觉得AsyncTask的处理上更灵活一些，在现在多核处理器满大街的情况下，充分的利用这个优势可以提高执行效率。

虽然Picasso没有提供根据CPU数量动态改变核心数，不过添加了根据当前网络状态改变CPU核心数的功能。

```text
void adjustThreadCount(NetworkInfo info) {
    if (info == null || !info.isConnectedOrConnecting()) {
      setThreadCount(DEFAULT_THREAD_COUNT);
      return;
    }
    switch (info.getType()) {
      case ConnectivityManager.TYPE_WIFI:
      case ConnectivityManager.TYPE_WIMAX:
      case ConnectivityManager.TYPE_ETHERNET:
        setThreadCount(4);
        break;
      case ConnectivityManager.TYPE_MOBILE:
        switch (info.getSubtype()) {
          case TelephonyManager.NETWORK_TYPE_LTE:  // 4G
          case TelephonyManager.NETWORK_TYPE_HSPAP:
          case TelephonyManager.NETWORK_TYPE_EHRPD:
            setThreadCount(3);
            break;
          case TelephonyManager.NETWORK_TYPE_UMTS: // 3G
          case TelephonyManager.NETWORK_TYPE_CDMA:
          case TelephonyManager.NETWORK_TYPE_EVDO_0:
          case TelephonyManager.NETWORK_TYPE_EVDO_A:
          case TelephonyManager.NETWORK_TYPE_EVDO_B:
            setThreadCount(2);
            break;
          case TelephonyManager.NETWORK_TYPE_GPRS: // 2G
          case TelephonyManager.NETWORK_TYPE_EDGE:
            setThreadCount(1);
            break;
          default:
            setThreadCount(DEFAULT_THREAD_COUNT);
        }
        break;
      default:
        setThreadCount(DEFAULT_THREAD_COUNT);
    }
  }

  private void setThreadCount(int threadCount) {
    setCorePoolSize(threadCount);
    setMaximumPoolSize(threadCount);
  }
```

那么在不同网络情况下改变核心线程数有什么作用呢？个人认为，随着网络情况越来越差，采用更小的核心线程数，可以更快的显示目标图片，毕竟在相同网速下，一个请求的加载速度肯定比多个请求的加载速度快。而在WIFI情况下，网速的限制比较小，多个网络请求可以带来更好的用户体验。

还有一个问题你想过吗：如何实现请求的优先级排序？考虑这样一个情景，如果滑动ListView，你最想哪一些图片先加载呢？当然是最后添加的请求先加载，因为这代表着用户当前关注的内容，所以这就需要维护一个根据优先级排序的集合，Picasso是如何实现的呢？

如果你有仔细看PicassoExecutorService的构造函数，就会发现缓存队列使用的是PriorityBlockingQueue这个数据结构，PriorityBlockingQueue的特点就是当元素添加的时候，会根据`Comparable.compareTo()`进行优先级排序。

但是添加到线程池的BitmapHunter并没有实现这个接口呀！

这是因为在`PicassoExecutorService.submit()`中进行了处理

PicassoExecutorService

```text
 @Override
  public Future<?> submit(Runnable task) {
    //进行了封装，使之可以根据优先级排序
    PicassoFutureTask ftask = new PicassoFutureTask((BitmapHunter) task);
    execute(ftask);
    return ftask;
  }

  private static final class PicassoFutureTask extends FutureTask<BitmapHunter>
      implements Comparable<PicassoFutureTask> {
    private final BitmapHunter hunter;

    public PicassoFutureTask(BitmapHunter hunter) {
      super(hunter, null);
      this.hunter = hunter;
    }

    @Override
    public int compareTo(PicassoFutureTask other) {
      Picasso.Priority p1 = hunter.getPriority();
      Picasso.Priority p2 = other.hunter.getPriority();
      return (p1 == p2 ? hunter.sequence - other.hunter.sequence : p2.ordinal() - p1.ordinal());
    }
  }
```

Priority属性在生成Request的时候被指定，一共有三个等级：

Picasso

```text
public enum Priority {
    LOW,
    NORMAL,
    HIGH
  }
```

当使用fetch\(\)发送任务时，默认等级为LOW，其他情况下默认优先级为NORMAL，如果你想指定一个请求的优先级，可以这样指定：

```text
 mPicasso.load(URL).priority(Picasso.Priority.HIGH).into(img);
```

但是这样指定的是总体上的优先级，线程集合中会存在很多相同优先级的请求，这个时候会根据sequence来确定最终的优先级。

而sequence的赋值发生在实例化BitmapHunter的过程中。

```text
private static final AtomicInteger SEQUENCE_GENERATOR = new AtomicInteger();
final int sequence;
BitmapHunter(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats, Action action,
      RequestHandler requestHandler) {
    this.sequence = SEQUENCE_GENERATOR.incrementAndGet();
    ...
  }
```

SEQUENCE\_GENERATOR是一个线程安全的Integer加减类，所以我们可以得出结论，如果两个请求的优先级相同，那么后创建的任务优先级会更高。

这个结论也验证了我前面提出的猜测，即最后添加的任务最先执行可以提供更好的用户体验。

扯得有点远了，下面再看一下BitmapHunter被添加到线程池之后的run\(\)到底做了什么。

## 网络加载图片是如何完成的？

在`BitmapHunter.run()`中完成了图片的获取操作。

```text
@Override public void run() {
    try {
             ...

      result = hunt();

      if (result == null) {
        dispatcher.dispatchFailed(this);
      } else {
        dispatcher.dispatchComplete(this);
      }
    } catch (Downloader.ResponseException e) {
        ...
    } finally {
      Thread.currentThread().setName(Utils.THREAD_IDLE_NAME);
    }
  }
```

主要的工作都在hunt\(\)完成，"hunt"意为搜寻、打猎，我们要开始打猎Bitmap了！

```text
 Bitmap hunt() throws IOException {
    Bitmap bitmap = null;
    //首先从内存中查找
    if (shouldReadFromMemoryCache(memoryPolicy)) {
      bitmap = cache.get(key);
      if (bitmap != null) {
        //统计Cache命中
        stats.dispatchCacheHit();
        loadedFrom = MEMORY;
        if (picasso.loggingEnabled) {
          log(OWNER_HUNTER, VERB_DECODED, data.logId(), "from cache");
        }
        return bitmap;
      }
    }

    data.networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;
    //根据不同的RequestHandler来从不同渠道获取数据
    RequestHandler.Result result = requestHandler.load(data, networkPolicy);
    if (result != null) {
      loadedFrom = result.getLoadedFrom();
      exifRotation = result.getExifOrientation();

      bitmap = result.getBitmap();
      if (bitmap == null) {
        InputStream is = result.getStream();
        try {
          bitmap = decodeStream(is, data);
        } finally {
          Utils.closeQuietly(is);
        }
      }
    }

    //如果需要对Bitmap进行转换则转换
    if (bitmap != null) {
      stats.dispatchBitmapDecoded(bitmap);
      if (data.needsTransformation() || exifRotation != 0) {
         //对转换操作加锁，如果存在多个需要转换的Bitmap，可以减少内存占用，避免OOM的发生
         //借鉴自Volley
        synchronized (DECODE_LOCK) {
          if (data.needsMatrixTransform() || exifRotation != 0) {
            bitmap = transformResult(data, bitmap, exifRotation);
          }
          if (data.hasCustomTransformations()) {
            bitmap = applyCustomTransformations(data.transformations, bitmap);
          }
        }
        if (bitmap != null) {
          stats.dispatchBitmapTransformed(bitmap);
        }
      }
    }

    return bitmap;
  }
```

加载的动作发生在`RequestHandler.load()`，不同的处理器有不同的实现方式。下面看一下网络获取器的实现。

```text
  @Override 
  public Result load(Request request, int networkPolicy) throws IOException {
    //用下载器下载图片
    Response response = downloader.load(request.uri, request.networkPolicy);
    if (response == null) {
      return null;
    }

    Picasso.LoadedFrom loadedFrom = response.cached ? DISK : NETWORK;
    //根据不同数据来创建Result
    Bitmap bitmap = response.getBitmap();
    if (bitmap != null) {
      return new Result(bitmap, loadedFrom);
    }

    InputStream is = response.getInputStream();
    if (is == null) {
      return null;
    }

    if (loadedFrom == DISK && response.getContentLength() == 0) {
      Utils.closeQuietly(is);
      throw new ContentLengthException("Received response with 0 content-length header.");
    }
    if (loadedFrom == NETWORK && response.getContentLength() > 0) {
      stats.dispatchDownloadFinished(response.getContentLength());
    }
    return new Result(is, loadedFrom);
  }
```

到此为止，一张图片已经加载完毕，等待显示了。

## Picasso是如何实现硬盘缓存的？

一般来说，图片加载框架都会有硬盘缓存，来减少流量消耗和加快加载速度，但是自始至终，我们都没有发现硬盘缓存的影子，那么Picasso是如何完成的硬盘缓存呢？

答案就在下面这段代码中。

```text
Response response = downloader.load(request.uri, request.networkPolicy);
```

图片资源的下载和本硬盘缓存都在这里完成。

UrlConnectionDownloader

```text
private static final String FORCE_CACHE = "only-if-cached,max-age=2147483647";

 @Override 
 public Response load(Uri uri, int networkPolicy) throws IOException {
     //只有在API>=14的版本上才能使用本地缓存
     //缓存地址是data/data/<package-name>/cache/picasso-cache
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
      installCacheIfNeeded(context);
    }

    HttpURLConnection connection = openConnection(uri);
    connection.setUseCaches(true);

    if (networkPolicy != 0) {
      String headerValue;
      //强制使用Cache
      if (NetworkPolicy.isOfflineOnly(networkPolicy)) {
        headerValue = FORCE_CACHE;
      } else {
        StringBuilder builder = CACHE_HEADER_BUILDER.get();
        builder.setLength(0);

        if (!NetworkPolicy.shouldReadFromDiskCache(networkPolicy)) {
          builder.append("no-cache");
        }
        if (!NetworkPolicy.shouldWriteToDiskCache(networkPolicy)) {
          if (builder.length() > 0) {
            builder.append(',');
          }
          builder.append("no-store");
        }

        headerValue = builder.toString();
      }

      connection.setRequestProperty("Cache-Control", headerValue);
    }

    int responseCode = connection.getResponseCode();
    if (responseCode >= 300) {
      connection.disconnect();
      throw new ResponseException(responseCode + " " + connection.getResponseMessage(),
          networkPolicy, responseCode);
    }

    long contentLength = connection.getHeaderFieldInt("Content-Length", -1);
    boolean fromCache = parseResponseSourceHeader(connection.getHeaderField(RESPONSE_SOURCE));

    return new Response(connection.getInputStream(), fromCache, contentLength);
  }
```

通过上面的源码可以知道，Picasso是通过HTML的Cache语义来实现本地缓存的。如果需要使用HttpUrlConnection实现缓存，运行版本必须在14及以上，因为HttpResponseCache是新添加的Cache类，而如果使用OkHttp则没有这个问题。

## Bitmap终于要显示出来了！

在`BitmapHunter.run()`中，在加载成功后，会调用`Diapatcher.dispatchComplete(BitmapHunter)`完成最后的回调。

```text
void dispatchComplete(BitmapHunter hunter) {
    handler.sendMessage(handler.obtainMessage(HUNTER_COMPLETE, hunter));
  }

  @Override 
  public void handleMessage(final Message msg) {
      switch (msg.what) {
          case HUNTER_COMPLETE: {
          BitmapHunter hunter = (BitmapHunter) msg.obj;
          dispatcher.performComplete(hunter);
          break;
  }

  void performComplete(BitmapHunter hunter) {
    //添加内存缓存
    if (shouldWriteToMemoryCache(hunter.getMemoryPolicy())) {
      cache.set(hunter.getKey(), hunter.getResult());
    }
    hunterMap.remove(hunter.getKey());
    //批量向主线程发送
    batch(hunter);
  }
```

这里有一个小地方需要注意，batch\(\)操作的功能是每隔200ms批量发送一次消息至主线程。

```text
private static final int BATCH_DELAY = 200; // ms
final List<BitmapHunter> batch;

 Dispatcher() {
    this.batch = new ArrayList<BitmapHunter>(4);
  }

private void batch(BitmapHunter hunter) {
    if (hunter.isCancelled()) {
      return;
    }
    batch.add(hunter);
    if (!handler.hasMessages(HUNTER_DELAY_NEXT_BATCH)) {
      handler.sendEmptyMessageDelayed(HUNTER_DELAY_NEXT_BATCH, BATCH_DELAY);
    }
  }
```

batch的默认长度为4，猜测这是经过测试得到的最大值，即200ms内最多完成的任务是4个，可能没有科学依据，但是实践出真知。

每隔200ms，就会将一批次最多4个BitmapHunter发送至主线程Handler。

```text
void performBatchComplete() {
    List<BitmapHunter> copy = new ArrayList<BitmapHunter>(batch);
    batch.clear();
    mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(HUNTER_BATCH_COMPLETE, copy));
    logBatch(copy);
  }
```

在主线程Handler中的处理则比较简单：

```text
static final Handler HANDLER = new Handler(Looper.getMainLooper()) {
    @Override public void handleMessage(Message msg) {
      switch (msg.what) {
        case HUNTER_BATCH_COMPLETE: {
          List<BitmapHunter> batch = (List<BitmapHunter>) msg.obj;
          for (int i = 0, n = batch.size(); i < n; i++) {
            BitmapHunter hunter = batch.get(i);
            hunter.picasso.complete(hunter);
          }
          break;
        }
       ...
      }
    }
  };
```

在`Picasso.complete()`中完成了图片的最终显示。

```text
void complete(BitmapHunter hunter) {
    Action single = hunter.getAction();
    List<Action> joined = hunter.getActions();

    boolean hasMultiple = joined != null && !joined.isEmpty();
    boolean shouldDeliver = single != null || hasMultiple;

    if (!shouldDeliver) {
      return;
    }

    Uri uri = hunter.getData().uri;
    Exception exception = hunter.getException();
    Bitmap result = hunter.getResult();
    LoadedFrom from = hunter.getLoadedFrom();

    if (single != null) {
      deliverAction(result, from, single);
    }

    if (hasMultiple) {
      //noinspection ForLoopReplaceableByForEach
      for (int i = 0, n = joined.size(); i < n; i++) {
        Action join = joined.get(i);
        deliverAction(result, from, join);
      }
    }

    if (listener != null && exception != null) {
      listener.onImageLoadFailed(this, uri, exception);
    }
  }
```

通过`Picasso.deliverAction()`会将结果分发至某一个Action，完成图片设置。

```text
 private void deliverAction(Bitmap result, LoadedFrom from, Action action) {
    if (action.isCancelled()) {
      return;
    }
    if (!action.willReplay()) {
      targetToAction.remove(action.getTarget());
    }
    if (result != null) {
      if (from == null) {
        throw new AssertionError("LoadedFrom cannot be null.");
      }
      action.complete(result, from);
    } else {
      action.error();
    }
  }
```

这里以ImageViewAction为例：

```text
@Override 
public void complete(Bitmap result, Picasso.LoadedFrom from) {
    if (result == null) {
      throw new AssertionError(
          String.format("Attempted to complete action with no result!\n%s", this));
    }

    //由于是若引用，所以需要判断Null
    ImageView target = this.target.get();
    if (target == null) {
      return;
    }

    Context context = picasso.context;
    boolean indicatorsEnabled = picasso.indicatorsEnabled;
    //完成ImageView的图片显示
    PicassoDrawable.setBitmap(target, context, result, from, noFade, indicatorsEnabled);
    if (callback != null) {
      callback.onSuccess();
    }
  }
```

很奇怪，最后没有直接设置ImageView，而是交给了PicassoDrawable，这个类是做什么的？

## PicassoDrawable是什么？

这要涉及到Picasso的一个小功能，使用下面的方式，可以在每张图片的左上角显示一个角标。

```text
Picasso.setIndicatorsEnabled(true);
```

红色表示网络加载，蓝色表示硬盘加载，绿色表示内存加载，使用这一个特性可以很方便的查看每张图片的加载途径。

那么这个功能是如何实现的呢？关键就在于PicassoDrawable,PicassoDrawable继承自BitmapDrawable，并重写onDraw\(\)。

```text
 @Override 
 public void draw(Canvas canvas) {

   ...

    if (debugging) {
      drawDebugIndicator(canvas);
    }
  }

   private void drawDebugIndicator(Canvas canvas) {
    DEBUG_PAINT.setColor(WHITE);
    Path path = getTrianglePath(new Point(0, 0), (int) (16 * density));
    canvas.drawPath(path, DEBUG_PAINT);

    DEBUG_PAINT.setColor(loadedFrom.debugColor);
    path = getTrianglePath(new Point(0, 0), (int) (15 * density));
    canvas.drawPath(path, DEBUG_PAINT);
  }

  private static Path getTrianglePath(Point p1, int width) {
    Point p2 = new Point(p1.x + width, p1.y);
    Point p3 = new Point(p1.x, p1.y + width);

    Path path = new Path();
    path.moveTo(p1.x, p1.y);
    path.lineTo(p2.x, p2.y);
    path.lineTo(p3.x, p3.y);

    return path;
  }
```

实现方式非常简单，就是画了两个Path，第二个Path小1dp，这样就出现了下图的效果。

![](http://i4.tietuku.com/ade90776cd1c09b1.png)

## 守护线程

Picasso内部还维护着一个守护线程，负责取消已经被GC回收的Target对应的请求。

```text
final ReferenceQueue<Object> referenceQueue;
private final CleanupThread cleanupThread;

private static class CleanupThread extends Thread {
    private final ReferenceQueue<Object> referenceQueue;
    private final Handler handler;

    CleanupThread(ReferenceQueue<Object> referenceQueue, Handler handler) {
      this.referenceQueue = referenceQueue;
      //主线程Handler
      this.handler = handler;
      //设置为守护线程，守护的是UI线程
      setDaemon(true);
      setName(THREAD_PREFIX + "refQueue");
    }

    @Override public void run() {
      Process.setThreadPriority(THREAD_PRIORITY_BACKGROUND);
      //轮训
      while (true) {
        try {
          //每隔1000ms，就从referenceQueue中取出一个即将被回收的引用
          RequestWeakReference<?> remove =
              (RequestWeakReference<?>) referenceQueue.remove(THREAD_LEAK_CLEANING_MS);
          Message message = handler.obtainMessage();
          //如果当前不为空，说明有对象要被回收
          if (remove != null) {
            //发送Message，主线程取消任务
            message.what = REQUEST_GCED;
            message.obj = remove.action;
            handler.sendMessage(message);
          } else {
            message.recycle();
          }
        } catch (InterruptedException e) {
          break;
        } catch (final Exception e) {
          handler.post(new Runnable() {
            @Override public void run() {
              throw new RuntimeException(e);
            }
          });
          break;
        }
      }
    }

    void shutdown() {
      interrupt();
    }
  }
```

首先这里用到了一个非常有用的数据结构ReferenceQueue，在生成Action的时候，Picasso在内部进行了封装：

```text
static class RequestWeakReference<M> extends WeakReference<M> {
    final Action action;

    public RequestWeakReference(Action action, M referent, ReferenceQueue<? super M> q) {
      super(referent, q);
      this.action = action;
    }
  }

 Action(Picasso picasso, T target, Request request, int memoryPolicy, int networkPolicy,
      int errorResId, Drawable errorDrawable, String key, Object tag, boolean noFade) {

    this.target =
        target == null ? null : new RequestWeakReference<T>(this, target, picasso.referenceQueue);
  }
```

我们可以发现，RequestWeakReference中连接着Action和Target，在实例化WeakRefrence时，如果传入ReferenceQueue，那么这个弱引用被GC回收️时，会被添加进ReferenceQueue，因此不断轮训可以获取到被GC的对象，在Handler中取消对应的任务。

```text
static final Handler HANDLER = new Handler(Looper.getMainLooper()) {
    @Override public void handleMessage(Message msg) {
      switch (msg.what) {
        case REQUEST_GCED: {
          Action action = (Action) msg.obj;
          action.picasso.cancelExistingRequest(action.getTarget());
          break;
        }

      }
    }
  };
```

## 总结

我们总算是将Picasso比较详细的学习了一遍，现在是总结时间，有总结才能有提高~

首先我们思考一下，一个通用的图片加载库应该有哪几部分组成？

我感觉至少有以下几部分：

1. Manager，指的就是Picasso，负责请求的发起、暂停、恢复、取消等操作
2. ThreadPool，指的就是PicassoExecutorService，负责线程数量的维护，合理的设置线程池参数会很大的影响到一个框架的整体实力
3. Diapatcher，指的就是分发器，负责任务的下发和管理，把一个BitmapHunter交给合适的RequestHandler
4. RequestHandler，指的就是处理器，不同的处理器有不同的加载策略，满足不同的需求
5. Cache，包括本地硬盘缓存和内存缓存，负责提高同一个请求的响应速度
6. Displayer，即显示器，这里指的就是PicassoDrawable或者说是ImageView

Ok，现在咱们已经有了这几部分了，那么整个图片加载流程也就很清楚了：使用Picasso发起一个图片加载请求，通过RequestCreator来控制Request的参数，比如设置错误图片、占位图片、优先级等，然后根据这些参数创建对应的Action，并且将Action添加到Dispatcher中等待分发，Dispatcher会根据Action的uri来选择不同的RequestHandler创建BitmapHunter，在这之后一个包含着各种参数的BitmapHunter就被无情的扔进了线程池中，然后会在BitmapHunter.run\(\)中根据不同的处理器来获取数据，最后又将结果通过Dispatcher和Handler传递到UI线程，将PicassoDrawable设置给ImageView，最终完成图片的显示工作。

除了学习到整个流程，还有什么收获吗？

我觉得是有的，Picasso在很多集合类的选择上是非常合适的，根据不同的需求选择不同的集合类，可以提高代码的质量。比如WeakReference的使用可以避免内存泄露，使用`Collections.unmodifiableList()`处理不可变List，使用PriorityBlockingQueue作为线程池的缓存可以按照优先级自动排序，对ArrayList的遍历采用for循环比迭代器更有效率、使用ReferenceQueue来取消无用任务等等，在这些地方都可以体现出Picasso代码的优秀。

除此之外，我们已经深入了解了Picasso的实现原理和代码细节，所以在使用的时候就可以发挥Picasso的最大功效和避免一些坑，比如：

* 使用fit\(\)之后就不能resize\(\)，否则会报错
* 可以使用Picasso.cancleTag\(\)避免无用请求和内存泄露
* 在ListView滑动的时候，可以监听滚动事件，使用Picasso.pauseTag\(\)来暂停加载，使用Picasso.resumeTag\(\)来恢复加载，避免ListView的卡顿
* 如果使用默认的网络加载，在API 14以下不支持硬盘缓存，换成OkHttp则全版本支持
* 使用Picasso.fetch\(\)可以预加载图片
* 可以使用Picasso.setIndicatorsEnabled\(true\)来便捷的查看图片来源
* 使用Picasso.with\(\).load\(URL\).get\(\)可以同步获取数据，但是没有内存缓存
* 使用resize\(\)可以压缩图片到指定尺寸
* 使用Picasso.with\(\).load\(URL\).transform\(\)可以对图片进行变换
* 当设置了图片的宽高\(使用fit\(\)、resize\(\)\)之后，会利用BitmapFactory.Options.inSampleSize和BitmapFactory.Options..inJustDecodeBounds对图片进行压缩
* 使用Picasso.getSnapshot\(\).dump\(\)可以获取到所有的缓存数据

```text
===============BEGIN PICASSO STATS ===============
Max Cache Size: 38347922
  Cache Size: 0
  Cache % Full: 0
  Cache Hits: 0
  Cache Misses: 1
Network Stats
  Download Count: 0
  Total Download Size: 0
  Average Download Size: 0
Bitmap Stats
  Total Bitmaps Decoded: 0
  Total Bitmap Size: 0
  Total Transformed Bitmaps: 0
  Total Transformed Bitmap Size: 0
  Average Bitmap Size: 0
  Average Transformed Bitmap Size: 0
===============END PICASSO STATS
```

至此，我们对Picasso的学习就算是基本完成了。

