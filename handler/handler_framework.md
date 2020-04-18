# FrameWork中哪些模块用到了Handler机制？

在FrameWork中很多重要的模块都可以看到Handler机制的身影，通过理解这些使用场景中Handler的作用，可以让我们更加深刻的理解Handler机制。

## Activity.runOnUiThread\(\)

在Activity中，使用`runOnUiThread()`可以很轻松的实现到UI线程的切换，其内部实现也是Handler机制。

framework/base/core/java/android/app/Activity.java

```text
final Handler mHandler = new Handler();

public final void runOnUiThread(Runnable action) {
        if (Thread.currentThread() != mUiThread) {
            mHandler.post(action);
        } else {
            action.run();
        }
    }
```

从代码中我们可以看到，Activity中其实是存在一个Handler成员变量的，但是由于访问权限是`default`，所以我们无法直接使用。而且`runOnUiThread()`不支持延时Runnable，所以如果想使用Handler的其他功能的话，我们还是需要自己单独定义Handler。

由于Handler的初始化发生在Activity被实例化的过程中，所以Handler绑定的线程与Activity被实例化的线程是相同的，而Activity被实例化发生在UI线程，所以我们的Runnable总是可以在UI线程执行。关于Activity被实例化的详细过程，将在下面的文章里进行介绍。

## AsyncTask的异步处理

AsyncTask是Android提供给我们的一个实现异步的类，`AsyncTask.doInBackground()`会在线程池中运行，而`AsyncTask.onPostExecute()`和`AsyncTask.onProgressUpdate()`则在UI线程中调用。在这个场景中需要进行工作线程与UI线程的切换，内部的实现机制就是Handler。

为了保证Handler在UI线程中被实例化，在`ActivityThread.main()`中就进行了这步操作。

```text
public static void main(String[] args) {
        Looper.prepareMainLooper();

        AsyncTask.init();

        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

```text
    private static final InternalHandler sHandler = new InternalHandler();

    //强制静态变量sHandler被初始化
    public static void init() {
        sHandler.getLooper();
    }

    private static class InternalHandler extends Handler {

        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

由上面代码我们很容易就可以看出，应用内所有AsyncTask的UI回调都是通过同一个静态变量sHandler实现的，而sHandler在UI线程中进行的实例化，即与UI线程进行了绑定，从而保证其handleMessage\(\)是在UI线程完成。

## ActivityThread与AMS的通信

ActivityThread与AMS的通信涉及到Binder机制，当要开启一个Activity时，Instrumentation会通过ActivityManagerProxy将数据通过Binder传递给AMS，当AMS协调工作完毕后，会通过ApplicationThreadProxy通知ActivityThread开启具体的Activity，但是由于这部分通信是在Binder线程池中，因此ActivityThread需要将线程切换至UI线程，这里就用到了Handler机制。

下面，将重点介绍AMS通过ApplicationThreadProxy通知ActivityThread开启新的Activity这个过程。

因为ApplicationThreadProxy代理的是ApplicationThread，所以具体的实现是在ApplicationThread。

```text
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, CompatibilityInfo compatInfo,
                IVoiceInteractor voiceInteractor, int procState, Bundle state,
                PersistableBundle persistentState, List<ResultInfo> pendingResults,
                List<Intent> pendingNewIntents, boolean notResumed, boolean isForward,
                ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);
            //ActivityClientRecord的数据准备操作
            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            updatePendingConfiguration(curConfig);
            //发送Message
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```

因为ApplicationThread是ActivityThread的内部类，所以具体的发送操作是在ActivityThread完成。

```text
public final class ActivityThread {

    final H mH = new H();

    private void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }

    private void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }

    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }

    private class H extends Handler {

        public static final int LAUNCH_ACTIVITY         = 100;

        public void handleMessage(Message msg) {

        switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
                } break;

                ...

            }
```

由于mH是在UI线程初始化，因此`H.handleMessage()`的执行线程为UI线程，从而完成了从Binder线程到UI线程的切换。

## Toast的显示与隐藏

如果想在子线程中显示Toast应该怎么做呢？必须在Thread中创建消息循环，否则就会报错`Can't create handler inside thread that has not called Looper.prepare()`。

```text
new Thread(new Runnable() {

            @Override
            public void run() {
                Looper.prepare();
                Toast.makeText(context,message).show();
                Looper.loop();
            }
        }).start();
```

为什么会这样呢？因为Toast的显示也需要Handler机制来完成线程的切换。

```text
public class Toast {

    final TN mTN;

    public Toast(Context context) {
        mContext = context;
        //实例化TN，TN负责与NotificationManager进行Binder通信
        mTN = new TN();
    }

    private static class TN extends ITransientNotification.Stub {

        final Runnable mShow = new Runnable() {
            @Override
            public void run() {
                //最终的Toast的显示和隐藏，是在handleShow()中通过WindowManager完成
                handleShow();
            }
        };

        final Runnable mHide = new Runnable() {
            @Override
            public void run() {
                handleHide();
                handleShow()
                mNextView = null;
            }
        };

        final Handler mHandler = new Handler();

        @Override
        public void show() {
            //当Toast确实需要显示的时候，NotificationManager会通过Binder调用show()
            //为了保证View的操作在UI线程，需要用Handler进行线程的切换
            mHandler.post(mShow);
        }

        @Override
        public void hide() {
            mHandler.post(mHide);
        }
    }
}
```

从上面的代码可以很容易看出，当Toast完成实例化的时候，就会完成TN的实例化，而在TN中的成员变量mHandler的实例化就是在Toast实例化的线程完成的，所以如果当前Toast所在线程没有绑定Looper的时候，就会抛出RuntimeException,这也是为什么在子线程中使用Toast时比如手动开启Looper循环的根本原因。

## 界面刷新

Android的单线程编程模式，决定了对View的界面操作都需要在UI线程执行，当我们调用`View.invalidate()`通知系统刷新界面时，最终会将这个请求传递到ViewRootImpl，具体流程我们会在后面的View机制中详细介绍，现在重点关注一下传递到ViewRootImpl之后的操作。

```text

```

