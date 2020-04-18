# Activity与Window是什么关系？

简单来说，它们之间的关系是：一个Activity中至少存在一个Window与之绑定，并由Window负责处理一些用户事件和系统事件。

Window与Activity的绑定操作发生在Activity初始化的时候。`Activity.attach()`是Activity被实例化之后第一个调用的方法，主要是进行成员变量的赋值和初始化操作。

framework/base/core/java/android/app/Activity.java

```text
private Window mWindow;
private WindowManager mWindowManager;
private Thread mUiThread;
private Instrumentation mInstrumentation;
private Application mApplication;
ActivityThread mMainThread;

final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, IVoiceInteractor voiceInteractor) {
        attachBaseContext(context);

        mFragments.attachActivity(this, mContainer, null);

        //初始化Window，Window是一个接口，实际类型为PhoneWindow
        mWindow = PolicyManager.makeNewWindow(this);
        //在这里设置Window的Callback，当调用setContentView()时会调用Activity.onContentChanged()
        //Activity.onContentChanged()的默认实现为空
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        //传说中的UI线程，赋值为当前线程，所以我们可以得出结论，Activity.attach()一定在UI线程被调用
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        //在一个App中默认只存在一个Instrumentation对象，所有Activity都指向同一个对象
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;

        ...

        mWindow.setWindowManager(...);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();

        ...

    }
```

在`Activity.attach()`中完成了Window和WindowManager的初始化，同时完成了Window回调函数的绑定，Activity就是这些这些回调函数的实现类。

framework/base/core/java/android/app/Activity.java

```text
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback {
        }
```

与Window相关的主要包括`Window.Callback`、`Window.OnWindowDismissedCallback`，下面将介绍这两个回调接口的作用。

接口`Window.Callback`定义在Window内部，其主要方法如下所示，完整方法请参考源码：

```text
public interface Callback {

        public boolean dispatchKeyEvent(KeyEvent event);

        public boolean dispatchKeyShortcutEvent(KeyEvent event);

        public boolean dispatchTouchEvent(MotionEvent event);

        public boolean dispatchTrackballEvent(MotionEvent event);

        public void onWindowAttributesChanged(WindowManager.LayoutParams attrs);

        public void onContentChanged();

        public void onWindowFocusChanged(boolean hasFocus);

        public void onAttachedToWindow();

        public void onDetachedFromWindow();

        ...

    }
```

下面我们抽取部分重要方法进行学习。

## Window.Callback

### onContentChanged\(\);

此回调发生在`PhoneWindow.setContentView()`、`PhoneWindow.addContentView()`。

frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java

```text
@Override
    public void setContentView(int layoutResID) {

        ...

        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```

在`getCallback()`中获取的是与此PhoneWindow绑定的Activity，因此最终会调用`Activity.onContentChanged()`。

```text
public void onContentChanged() {}
```

实际上，当我们调用`Activity.setContentView()`时，最终调用的是`PhoneWindow.setContentView()`，因此你如果想在布局文件设置完之后立即做一些操作，可以通过覆写这个方法完成。

不过一般来说，直接写在`Activity.onCreate()`中得到的效果是完全一样的：

```text
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_all);

        ...doSomeThing...

    }
```

### dispatchKeyEvent\(\)与dispatchTouchEvent\(\)

这是两个非常重要的方法，输入操作\(键盘输入与触摸事件\)的每一事件都会经过这些方法，然后再向下分发。

为了更好的说明事件的触发时机，下面将简单介绍一下Touch事件的产生及传递至Activity的流程。

当我们手机触摸屏幕时，硬件设备会感知到这些操作，并且将这个操作转化成一系列的输入事件，由于这个过程与我们开发无关，所以这里不深入研究。

最终Touch事件会传递到我们的FrameWork层，但是在后续的一些处理上，不同版本之间有细微的差别。

在早期的版本上，输入事件会传递到WMS，然后由WMS通过`ViewRootImpl::W`对输入事件进行分发，W是`IWindow.Stub`的子类，因此这种方式是通过Binder通信完成的。

frameworks/base/core/java/android/view/ViewRootImpl.java

```text
static class W extends IWindow.Stub {}
```

由于Binder通信是运行在Binder线程池，因此如果想实现后续的事件分发，需要将事件转发至UI线程，而这个操作就是由Handler机制完成的。在这之后，Touch事件就由WMS分发至获得焦点的`PhoneWindow::DecorView`。

```text
public void dispatchPointer(MotionEvent event, long eventTime,
            boolean callWhenDone) {
        Message msg = obtainMessage(DISPATCH_POINTER);
        msg.obj = event;
        msg.arg1 = callWhenDone ? 1 : 0;
        sendMessageAtTime(msg, eventTime);
    }
```

通过这种处理就可以完成输入事件的分发，但是以笔者所研究的5.0系统来说，IPC通信由Binder变成了Socket方式。

在`ViewRootImpl.setView()`中，完成了ViewRootImpl与WMS的连接，同时也完成了输入事件Socket通道的构建。

```text
final W mWindow;

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                ...

                int res;
                requestLayout();
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                        //完成InputChannel初始化
                    mInputChannel = new InputChannel();
                }
                try {
                    //通过Binder通信，通知WMS添加Window，同时把W和InputChannel传递给WMS
                    //至此，WMS就可以利用W完成Binder通信，通过InputChannel构建App与InputManager的Socket通信
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                } catch (RemoteException e) {
                    ...
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }

                ...

                if (mInputChannel != null) {
                    if (mInputQueueCallback != null) {
                        mInputQueue = new InputQueue();
                        mInputQueueCallback.onInputQueueCreated(mInputQueue);
                    }
                    //完成ImputManager与InputEventReceiver的绑定，后续事件都会调用InputEventReceiver的回调方法
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }

                ...

            }
        }
    }
```

在上述过程中，WMS拿到了W对象，从而可以通过W控制ViewRootImpl完成一些操作，而ViewRootImpl则通过Session控制WMS一些函数的调用，这样就构建了双向的Binder通信。

新版本与老版本在输入事件的处理上的不同在于，老版本是WMS通过Binder与App通信，而新版本则是ImputManager通过Socket完成IPC通信，虽然在IPC通信的实现方式上有所差异，但是整体上的CS架构是没有变化的。

当InputManager接收到输入事件时，会通过InputChannel调用`InputEventReceiver.onInputEvent()`，实际的实例化对象是`WindowInputEventReceiver`。

```text
 final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            super(inputChannel, looper);
        }

        @Override
        public void onInputEvent(InputEvent event) {
            enqueueInputEvent(event, this, 0, true);
        }

        @Override
        public void onBatchedInputEventPending() {
            if (mUnbufferedInputDispatch) {
                super.onBatchedInputEventPending();
            } else {
                scheduleConsumeBatchedInput();
            }
        }

        @Override
        public void dispose() {
            unscheduleConsumeBatchedInput();
            super.dispose();
        }
    }
```

因此会调用`ViewRootImpl.enqueueInputEvent()`

```text
void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {

        ...

        if (processImmediately) {
            doProcessInputEvents();
        } else {
            scheduleProcessInputEvents();
        }
    }
```

由于`processImmediately`为true，因此会调用`ViewRootImpl.doProcessInputEvents()`，在这里面进行了事件的分发。

```text
void doProcessInputEvents() {
        // 将所有Touch事件分发至消息队列中
        while (mPendingInputEventHead != null) {

            ...

            deliverInputEvent(q);
        }
    }
```

由于在后续`ViewRootImpl.deliverInputEvent()`的事件分发的过程中非常繁琐，而且与我们日常开发工作关系并不大，因此我以查看方法栈的方式来查看Touch事件的分发过程。**这也是一种很好的学习方法，如果在研究源码过程中，很难理解某一个模块的具体处理思路，那么可以使用这种方式来理清程序的走向，最终得出有用的结论。**

下面显示的是Touch事件被分发至`Activity.dispatchTouchEvent()`的过程。

![](http://i4.tietuku.com/c5557f979d72a6c1.png)

由上图我们可以看出，经过一系列的`InputStage.forward()`、`InputStage.apply()`、`InputStage.deliver()`操作之后，事件最终被传递到了`DecorView.dispatchTouchEvent()`，而在接下来的传递过程中，就用到了`Window.Callback`。

frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java

```text
private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
            final Callback cb = getCallback();
            return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
                    : super.dispatchTouchEvent(ev);
        }
}
```

这里会调用`Activity.dispatchTouchEvent()`。

framework/base/core/java/android/app/Activity.java

```text
public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

而`getWindow().superDispatchTouchEvent(ev)`实际调用的是`DecorView.superDispatchTouchEvent(event)`。

```text
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

而DecorView作为一个ViewGroup,实际调用的是ViewGroup.dispatchTouchEvent\(event\)。

```text
public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```

至此，Touch事件就从硬件输入设备传递到了FrameWork层的View Tree中，接下面Touch事件将在View中进行传递，这一部分我将在**View机制章节**进行介绍。

这个时候我们会发现，从DecorView将Touch事件通过`Window.Callback`传递给Activity之后，最终还是回到了DecorView并由其优先处理，为什么会这么做呢？

我觉得，通过这种方式可以把事件传递的生杀大权交给Activity,Activity就像是三峡大坝，Touch事件就像是长江中的鱼，只有三峡大坝开闸放水的时候，鱼儿才能东游入海。

同样的，如果Activity不允许View Tree处理任何Touch事件，那么可以直接重写`Activity.dispatchTouchEvent()`，然后整个世界就安静了~

```text
public class TouchActivity extends Activity {

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        return false;
    }
}
```

如果整个View Tree都不想对Touch事件进行处理的话，`Activity.getWindow().superDispatchTouchEvent(ev)`就会返回false，最终`Activity.onTouchEvent()`就会被调用。

当然，由于所有的Touch事件都会经过`Activity.dispatchTouchEvent()`，因此也可以在这个方法里面进行数据埋点操作，但是要注意不要在这里进行频繁的对象生成及计算操作，以免造成界面的卡顿。

### onWindowFocusChanged\(\)

该方法的调用时机为：当当前Window的输入焦点状态发生变化时。

主要包括以下场景：

* 首次进入Activity，在布局对用户可见且获取到输入焦点时
* 当前Activity被另外一个Activity遮挡时
* 点击Home按键，当前Activity进入后台时
* 显示系统Window时，如下拉StatusBar显示快捷操作菜单栏或显示系统通知栏时
* 在Activity中显示可以获取到输入焦点的子Window时，如Dialog

onWindowFocusChanged\(\)的调用是由WMS控制的，这里同样涉及到Binder跨进程通信和Handler机制，从这里也可以看出来Handler机制可以说是无处不在，深刻的理解Handler机制的作用和使用场景可以让我们更好的理解FrameWork。

下面我会介绍onWindowFocusChanged\(\)的调用过程，在这之前先给出方法调用栈。

![](http://i4.tietuku.com/ef9112c49af98d72.png)

从前文我们知道，WMS通过`ViewRootImpl::W`与ViewRootImpl进行Binder通信，从上面的调用栈我们也可以看出，最终执行是在`ViewRootImpl::ViewRootHandler`中，下面我们将从源码角度跟踪整个方法调用。

WMS会调用`W.windowFocusChanged()`来通知ViewRootImpl焦点变化。

frameworks/base/core/java/android/view/ViewRootImpl.java

```text
static class W extends IWindow.Stub {

public void windowFocusChanged(boolean hasFocus, boolean inTouchMode) {
            final ViewRootImpl viewAncestor = mViewAncestor.get();
            if (viewAncestor != null) {
                viewAncestor.windowFocusChanged(hasFocus, inTouchMode);
            }
        }
}
```

在W中调用了`ViewRootImpl.windowFocusChanged()`

```text
public void windowFocusChanged(boolean hasFocus, boolean inTouchMode) {
        Message msg = Message.obtain();
        msg.what = MSG_WINDOW_FOCUS_CHANGED;
        msg.arg1 = hasFocus ? 1 : 0;
        msg.arg2 = inTouchMode ? 1 : 0;
        mHandler.sendMessage(msg);
    }
```

在这里利用Handler将消息发送至UI线程，最终会在ViewRootHandler中进行处理，在早期的源码中，ViewRootHandler的作用是由ViewRootImpl来完成的，因为ViewRootImpl继承自Handler，但是在新版本中对职责进行了划分，将Handler的部分单独抽取到内部类中，这种处理方式一方面是为了应对越来越复杂的需求，另一方面则是通过功能模块的划分，使得代码更好的符合单一职责原则，有利于代码的理解和维护。

```text
final class ViewRootHandler extends Handler {

     @Override
        public void handleMessage(Message msg) {

             switch (msg.what) {

                ...

                case MSG_WINDOW_FOCUS_CHANGED: {
                    if (mAdded) {
                        boolean hasWindowFocus = msg.arg1 != 0;
                        mAttachInfo.mHasWindowFocus = hasWindowFocus;

                        ...

                        }

                        ...

                        if (mView != null) {

                            ...

                            //mView就是DecorView
                            mView.dispatchWindowFocusChanged(hasWindowFocus);
                            mAttachInfo.mTreeObserver.dispatchOnWindowFocusChange(hasWindowFocus);
                        }
                    }

                    ...

                    break;
             }
        }
}
```

由上面的代码可知，最终WMS的命令交给了`DecorView.dispatchWindowFocusChanged()`来完成，与此同时也完成了Binder线程到UI线程的转换。由于DecorView继承自FrameLayout，且这两者都未重写此方法，所以最终调用的是`Viewgroup.dispatchWindowFocusChanged()`。

frameworks/base/core/java/android/view/ViewGroup.java

```text
@Override
    public void dispatchWindowFocusChanged(boolean hasFocus) {
        super.dispatchWindowFocusChanged(hasFocus);
        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            children[i].dispatchWindowFocusChanged(hasFocus);
        }
    }
```

由此可以，首先调用`super.dispatchWindowFocusChanged(hasFocus)`，然后将焦点变化事件往子类进行分发，而`super.dispatchWindowFocusChanged(hasFocus)`调用的是ViewGroup父类的方法，即View的对应方法。

frameworks/base/core/java/android/view/View.java

```text
public void dispatchWindowFocusChanged(boolean hasFocus) {
        onWindowFocusChanged(hasFocus);
    }
```

View的处理非常简单，就是调用了`View.onWindowFocusChanged(hasFocus)`，而我们这里则是DecorView的对应方法，因为DecorView重写了此方法。

frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java

```text
 @Override
        public void onWindowFocusChanged(boolean hasWindowFocus) {
            super.onWindowFocusChanged(hasWindowFocus);

            // 如果当前Window失去焦点，则关闭系统菜单
            if (!hasWindowFocus && mPanelChordingKey != 0) {
                closePanel(FEATURE_OPTIONS_PANEL);
            }

            //在这里实际调用了Activity.onWindowFocusChanged()
            final Callback cb = getCallback();
            if (cb != null && !isDestroyed() && mFeatureId < 0) {
                cb.onWindowFocusChanged(hasWindowFocus);
            }
        }
```

OK，现在我们再来总结一下onWindowFocusChanged调用过程：

* WMS负责调度所有的Window，在合适的时机通过ViewRootImpl::W通知ViewRootImpl
* ViewRootImpl收到事件，通过Handler发送Message，由ViewRootHandler进行处理
* ViewRootHandler调用了DecorView.dispatchWindowFocusChanged\(\)
* DecorView.dispatchWindowFocusChanged\(\)中调用了DecorView.onWindowFocusChanged\(\)
* 通过getCallback\(\)获取到Activity，并调用Activity.onWindowFocusChanged\(\)，在这之后，调用View Tree中所有View的dispatchWindowFocusChanged\(\)事件，完成焦点改变事件的分发

如果你有在进入Activity之后，立即获取某一个View的宽高属性需求的话，可以通过Activity.onWindowFocusChanged\(\)来完成，当该方法被调用时，保证View的宽高属性已被正确赋值。但是这个事件在Activity的一个生命周期中可以被调用多次，并且与Activity的生命周期方法并无对应关系。

一个会造成此方法被调用，而Activity的生命周期方法不被触发的典型场景是**下拉系统通知栏**。

## OnWindowDismissedCallback接口介绍

OnWindowDismissedCallback接口是在较新版本中才添加的，同时被添加的还有自定义控件`SwipeDismissLayout`和新的Window标志位`FEATURE_SWIPE_TO_DISMISS`\(API 20\)，主要实现的功能是**侧滑关闭Activity**。

通过以下的设置，可以很容易的对Activity添加侧滑关闭特性。

```text
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        getWindow().requestFeature(Window.FEATURE_SWIPE_TO_DISMISS);
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_swipe_close);
    }
```

下面将介绍侧滑关闭功能的实现思路和局限性。

如果我们设置了`Window.FEATURE_SWIPE_TO_DISMISS`，那么通过`PhoneWindow.generateLayout()`生成布局界面的时候将对这个标志位进行特殊处理。

```text
 protected ViewGroup generateLayout(DecorView decor) {

    ...    

    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        registerSwipeCallbacks();
    }

    ...

 }

 private void registerSwipeCallbacks() {
        //设置标志位之后，content的实现为SwipeDismissLayout
        SwipeDismissLayout swipeDismiss =
                (SwipeDismissLayout) findViewById(R.id.content);
        swipeDismiss.setOnDismissedListener(new SwipeDismissLayout.OnDismissedListener() {
            @Override
            public void onDismissed(SwipeDismissLayout layout) {
                //如果根据手势判断用户想侧滑关闭Activity
                //那么这里将调用Activity.onWindowDismissed()
                dispatchOnWindowDismissed();
            }
        });
        swipeDismiss.setOnSwipeProgressChangedListener(
                new SwipeDismissLayout.OnSwipeProgressChangedListener() {
                    private static final float ALPHA_DECREASE = 0.5f;
                    private boolean mIsTranslucent = false;
                    @Override
                    public void onSwipeProgressChanged(
                            SwipeDismissLayout layout, float progress, float translate) {
                        //通过当前滑动的距离和百分比来改变Window的位移与透明度，完成Window的移动和变暗效果
                        WindowManager.LayoutParams newParams = getAttributes();
                        //设置位移
                        newParams.x = (int) translate;
                        //设置透明度
                        newParams.alpha = 1 - (progress * ALPHA_DECREASE);
                        setAttributes(newParams);

                        int flags = 0;
                        //位移为0的时候，设置全屏
                        if (newParams.x == 0) {
                            flags = FLAG_FULLSCREEN;
                        } else {
                        //否则设置标志位FLAG_LAYOUT_NO_LIMITS
                            flags = FLAG_LAYOUT_NO_LIMITS;
                        }
                        setFlags(flags, FLAG_FULLSCREEN | FLAG_LAYOUT_NO_LIMITS);
                    }

                    @Override
                    public void onSwipeCancelled(SwipeDismissLayout layout) {
                        WindowManager.LayoutParams newParams = getAttributes();
                        //滑动取消的时候，初始化Window的属性，使之回到原点，透明度为1
                        newParams.x = 0;
                        newParams.alpha = 1;
                        setAttributes(newParams);
                        setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN | FLAG_LAYOUT_NO_LIMITS);
                    }
                });
    }
```

通过上面的处理，我们可以很清楚的看到实现思路：通过控制Window的布局属性来完成Window的移动和透明度的改变。

但是这里有个地方不好理解，那就是为什么每次改变属性之后需要设置FLAG\_FULLSCREEN和FLAG\_FULLSCREEN的标志位？

在测试过程中我发现，这两种标志位产生的作用就是

1. 不做任何修改，则在取消滑动之后，屏幕显示方式变成了全屏
2. 如果只保留FLAG\_LAYOUT\_NO\_LIMITS，屏幕显示效果也是全屏
3. 如果把两个Flag都去掉，则Window只有透明度的变化，位置并未发生变化

最后这一点通过参考API文档后就知道原因了。

```text
/** Window flag: allow window to extend outside of the screen. */
public static final int FLAG_LAYOUT_NO_LIMITS   = 0x00000200;
```

这个Flag控制着Window是否能够超出Screen，因此如果不设置这个标志位的话，全屏状态的Window并不能发生位移。

通过这种方式可以实现Window的侧滑关闭功能，但是在滑动过程和取消滑动之后会改变原始的屏幕显示状态，这显然是不能完成我们的需求的，那么有没有其他好办法呢？

后来我想，能否通过Theme设置Window的背景颜色为透明，然后利用scrollTo\(\)来控制Activity中DecorView的移动实现呢？

经过测试，这样做确实可以避免屏幕状态的改变，但是也存在一个问题，那就是在一些机型上面会出现黑屏的现象，侧滑过程中不能显示后面的Activity的界面。对于出现这一问题的原因，我猜测是由于Surface的颜色造成的。

这种侧滑关闭的效果在IOS平台上比较常见，在Android平台上能否更加快捷的实现这种效果呢？

有Github，没有什么是不可能的。

开源项目[SwipeBackLayout](https://github.com/ikew0ng/SwipeBackLayout)可以实现这个效果，在实现过程中也遇到了黑屏的问题，但是通过反射解决这个问题：

```text
    /**
     * Calling the convertToTranslucent method on platforms before Android 5.0
     */
    public static void convertActivityToTranslucentBeforeL(Activity activity) {
        try {
            Class<?>[] classes = Activity.class.getDeclaredClasses();
            Class<?> translucentConversionListenerClazz = null;
            for (Class clazz : classes) {
                if (clazz.getSimpleName().contains("TranslucentConversionListener")) {
                    translucentConversionListenerClazz = clazz;
                }
            }
            Method method = Activity.class.getDeclaredMethod("convertToTranslucent",
                    translucentConversionListenerClazz);
            method.setAccessible(true);
            method.invoke(activity, new Object[] {
                null
            });
        } catch (Throwable t) {
        }
    }

    /**
     * Calling the convertToTranslucent method on platforms after Android 5.0
     */
    private static void convertActivityToTranslucentAfterL(Activity activity) {
        try {
            Method getActivityOptions = Activity.class.getDeclaredMethod("getActivityOptions");
            getActivityOptions.setAccessible(true);
            Object options = getActivityOptions.invoke(activity);

            Class<?>[] classes = Activity.class.getDeclaredClasses();
            Class<?> translucentConversionListenerClazz = null;
            for (Class clazz : classes) {
                if (clazz.getSimpleName().contains("TranslucentConversionListener")) {
                    translucentConversionListenerClazz = clazz;
                }
            }
            Method convertToTranslucent = Activity.class.getDeclaredMethod("convertToTranslucent",
                    translucentConversionListenerClazz, ActivityOptions.class);
            convertToTranslucent.setAccessible(true);
            convertToTranslucent.invoke(activity, null, options);
        } catch (Throwable t) {
        }
    }
```

当然这个项目的意义不仅于此，从这个项目还可以学习到如何尽量少的侵入代码，如何处理DecorView、Window的背景色等等。具体的实现请参照项目源码，这里不再赘述。

当然，上面扯了这么多，和OnWindowDismissedCallback接口并好像没有太大关系...

话说前面说到，当滑动动作满足一定条件时，会调用`PhoneWindow.dispatchOnWindowDismissed()`，实际调用的是Activity.onWindowDismissed\(\)。

frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java

```text
public final void dispatchOnWindowDismissed() {
        if (mOnWindowDismissedCallback != null) {
            mOnWindowDismissedCallback.onWindowDismissed();
        }
    }
```

在Activity中的处理则更简单粗暴：

```text
 @Override
    public void onWindowDismissed() {
        finish();
    }
```

至此，滑动退出的整个实现思路就算是基本完成了，OnWindowDismissedCallback接口就是为了完成这一个功能而出现的。

## 总结

在前面我们简单的介绍了Actvity实现的与Window相关的的部分接口的调用时机和作用，下面我们再来探讨一下Activity与Window的关系，便于我们更好的理解Android系统的设计理念。

Activity作为唯一可以直接与用户打交道的组件，其重要性不言而喻。

Activity主要负责下面这些重要功能：

1. UI界面的显示
2. 输入事件的分发与处理
3. View Tree的状态存储与恢复
4. Fragment状态的维护
5. Activity的开启与关闭
6. 生命周期方法的处理

从这些方面可以看出Activity包含了Android中很多的重要事件和功能，但是整个Activity的源码只有6000多行，与View源代码的20000多行相比，真是不值得一提。

那么Activity是如何在这么少的代码量的基础之上完成这么多功能的呢？

我觉得最关键的是依靠**类职责的合理划分**和**事件传递模型**完成的。

下面我将就这两方面谈下我的想法。

### 类职责的合理划分

这个很好理解，就是当业务变得越来越负责的时候，通过对类职责的合理划分，让某个类更专注于它自身的功能。

在上面我虽然提到了Activity的很多功能，但是实际上，这些功能Activity都没有亲自进行处理，而是将具体操作分发给了更具体的类来完成。

比如UI界面的显示、输入事件的分发与处理、View Tree的状态存储与恢复都是由View机制来处理的，这一些内容我将在View机制进行更加详尽的介绍。

再比如Fragment状态的维护，是由FragmentManagerImpl来处理的，Activity的开启是由Instrumentation来具体操作的，生命周期方法是由AMS来控制的。

因此，从这个角度来说，Activity更像是综合了很多功能的一个管理类，就像是UIL图片加载框架里面的ImageLoader，它是面向开发者的一个入口，使得开发工作更加简单，但是主要工作并不是它来主导完成的，而这一切的基石就是对类职责的合理划分。

从这一点我们也可以知道，虽然我们在写代码时都会赋予每个类某些功能，但是更合理的职责划分，将给我们的代码结构带来莫大的好处。

### 事件传递模型

事件传递模型解决的问题是：如何更加高效合理的将事件分发下去，并进行处理。

如何找出Touch事件的触发者？View Tree的状态如何进行存储和恢复？如何确定View的大小、位置并进行绘制？

这些问题都值得我们去思考，而Android系统采用的事件传递模型无疑是一种很好的处理方式。

关于View的Touch事件的传递过程，测量、定位、绘制的具体流程我将在**View机制**一章中详细介绍，这里以View Tree的状态存储和恢复为例简单的介绍这种事件传递模型。

我们都知道，如果当前Activity在意外情况下被遮挡或者是摧毁的话，系统会调用`Activity.onSaveInstanceState()`完成界面的数据备份操作，下面我将从源码角度介绍这个过程。

framework/base/core/java/android/app/Activity.java

```text
 protected void onSaveInstanceState(Bundle outState) {
        outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
        Parcelable p = mFragments.saveAllState();
        if (p != null) {
            outState.putParcelable(FRAGMENTS_TAG, p);
        }
        getApplication().dispatchActivitySaveInstanceState(this, outState);
    }
```

从上面的代码我们看出，Activity主要对以下数据进行了备份

* View Tree
* Fragment

Activity并没有做太多操作，而是将View的备份工作交给了PhoneWindow，将Fragment的备份操作交给了FragmentManagerImpl，下面主要关注View的备份过程。

frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java

```text
@Override
    public Bundle saveHierarchyState() {
        //如果当前没有布局，则直接返回
        Bundle outState = new Bundle();
        if (mContentParent == null) {
            return outState;
        }

        SparseArray<Parcelable> states = new SparseArray<Parcelable>();
        //对View状态进行保存
        mContentParent.saveHierarchyState(states);
        //添加到Bundle中
        outState.putSparseParcelableArray(VIEWS_TAG, states);

        // 保存当前获取到焦点的View的id
        View focusedView = mContentParent.findFocus();
        if (focusedView != null) {
            if (focusedView.getId() != View.NO_ID) {
                outState.putInt(FOCUSED_ID_TAG, focusedView.getId());
            } else {
                if (false) {
                    Log.d(TAG, "couldn't save which view has focus because the focused view "
                            + focusedView + " has no id.");
                }
            }
        }

        // 保存系统菜单的状态
        SparseArray<Parcelable> panelStates = new SparseArray<Parcelable>();
        savePanelState(panelStates);
        if (panelStates.size() > 0) {
            outState.putSparseParcelableArray(PANELS_TAG, panelStates);
        }

        //保存ActionBar的状态
        if (mDecorContentParent != null) {
            SparseArray<Parcelable> actionBarStates = new SparseArray<Parcelable>();
            mDecorContentParent.saveToolbarHierarchyState(actionBarStates);
            outState.putSparseParcelableArray(ACTION_BAR_TAG, actionBarStates);
        }

        return outState;
    }
```

从上面的代码可以看出，在PhoneWindow中，不仅仅保存了View Tree的状态，还保存了焦点View、系统菜单、ActionBar等数据。

mContentParent是一个ViewGroup，ViewGroup未重写saveHierarchyState\(\)，所以具体的操作是由View完成的。

frameworks/base/core/java/android/view/View.java

```text
public void saveHierarchyState(SparseArray<Parcelable> container) {
        dispatchSaveInstanceState(container);
    }

protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
        if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
            mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
            Parcelable state = onSaveInstanceState();
            if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
                throw new IllegalStateException(
                        "Derived class did not call super.onSaveInstanceState()");
            }
            if (state != null) {
                container.put(mID, state);
            }
        }
    }
```

但是ViewGroup重写了dispatchSaveInstanceState\(\)，你猜猜它里面是怎么处理的？我猜的话，肯定是遍历**Child View，然后分别调用**`View.dispatchSaveInstanceState()`。

是的，你猜对了。

```text
@Override
    protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
        super.dispatchSaveInstanceState(container);
        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            View c = children[i];
            if ((c.mViewFlags & PARENT_SAVE_DISABLED_MASK) != PARENT_SAVE_DISABLED) {
                c.dispatchSaveInstanceState(container);
            }
        }
    }
```

由此便完成了整个View Tree的状态存储工作。

由此可以看出，如果你对Android的事件分发机制有所了解之后，当再次面对类似的机制时，你就可以八九不离十的猜到系统的实现机制，这也是学习Android FrameWork层的一个很重要的作用。

最后来一句话总结Activity与Window的关系：**Activity是面向开发者的界面管理类，负责View Tree的显示、存储与恢复工作，是输入事件传递的枢纽，可以控制输入事件的分发，并且负责响应Window状态变化，而这一切的工作都是委托Window来完成的。**

至此，我们对Activity于Window的关系算是有一个基本了解了。

