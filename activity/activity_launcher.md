# Launcher是如何实现开启一个App的？

## Launcher是如何实现开启一个App的？

虽然已经学习Android很长时间了，但是下面这些问题的答案你是否都清楚呢？

* Launcher到底是什么神奇的东西？
* 一个App是怎么启动起来的？
* App的程序入口到底是哪里？
* Binder是什么？他是如何进行IPC通信的？
* Activity生命周期到底是被谁管理的？如何调用的？
* 等等...

下面我将结合着源码，来寻找以上这些问题的答案。

### Launcher是什么？什么时候启动的？

当我们点击手机桌面上的图标时，App就开始启动了。但是，你有没有思考过Launcher的本质是什么？

Launcher本质上是一个App，而与用户进行交互的则是一个Activity。

packages/apps/Launcher2/src/com/android/launcher2/Launcher.java

```text
public final class Launcher extends Activity
        implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks,
                   View.OnTouchListener {
                   }
```

Launcher实现了点击、长按等回调接口，来接收用户的输入。既然是普通的App，那么我们的开发经验在这里就仍然适用，比如，我们点击图标的时候，是怎么开启的应用呢？如果是你，你怎么做这个功能呢？捕捉图标点击事件，然后startActivity\(\)发送对应的Intent请求！

是的，Launcher也是这么做的！

那么到底是处理的哪个对象的点击事件呢？既然Launcher是App，并且有Activity，那么肯定有布局文件，下面就是布局文件launcher.xml

```text
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:launcher="http://schemas.android.com/apk/res/com.android.launcher"
    android:id="@+id/launcher">

    <com.android.launcher2.DragLayer
        android:id="@+id/drag_layer"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:fitsSystemWindows="true">

        <include
            android:id="@+id/dock_divider"
            layout="@layout/workspace_divider"
            android:layout_marginBottom="@dimen/button_bar_height"
            android:layout_gravity="bottom" />

        <include
            android:id="@+id/paged_view_indicator"
            layout="@layout/scroll_indicator"
            android:layout_gravity="bottom"
            android:layout_marginBottom="@dimen/button_bar_height" />

        <!-- The workspace contains 5 screens of cells -->
        <com.android.launcher2.Workspace
            android:id="@+id/workspace"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:paddingStart="@dimen/workspace_left_padding"
            android:paddingEnd="@dimen/workspace_right_padding"
            android:paddingTop="@dimen/workspace_top_padding"
            android:paddingBottom="@dimen/workspace_bottom_padding"
            launcher:defaultScreen="2"
            launcher:cellCountX="@integer/cell_count_x"
            launcher:cellCountY="@integer/cell_count_y"
            launcher:pageSpacing="@dimen/workspace_page_spacing"
            launcher:scrollIndicatorPaddingLeft="@dimen/workspace_divider_padding_left"
            launcher:scrollIndicatorPaddingRight="@dimen/workspace_divider_padding_right">

            <include android:id="@+id/cell1" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell2" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell3" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell4" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell5" layout="@layout/workspace_screen" />
        </com.android.launcher2.Workspace>

    ...ignore some code...

    </com.android.launcher2.DragLayer>
</FrameLayout>
```

为了方便查看，删除了很多代码，从上面这些我们应该可以看出一些东西来：Launcher大量使用**include**标签来实现界面的复用，而且定义了很多的自定义控件实现界面效果，dock\_divider从布局的参数声明上可以猜出，是底部操作栏和上面图标布局的分割线，而paged\_view\_indicator则是页面指示器。

当然，我们最关心的是Workspace这个布局，因为注释里面说在这里面包含了5个屏幕的单元格，想必你也猜到了，这个就是在首页存放我们图标的那五个界面\(不同的ROM会做不同的DIY，数量不固定\)。

接下来，我们应该打开workspace\_screen布局，看看里面有什么。

workspace\_screen.xml

```text
<com.android.launcher2.CellLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:launcher="http://schemas.android.com/apk/res/com.android.launcher"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:paddingStart="@dimen/cell_layout_left_padding"
    android:paddingEnd="@dimen/cell_layout_right_padding"
    android:paddingTop="@dimen/cell_layout_top_padding"
    android:paddingBottom="@dimen/cell_layout_bottom_padding"
    android:hapticFeedbackEnabled="false"
    launcher:cellWidth="@dimen/workspace_cell_width"
    launcher:cellHeight="@dimen/workspace_cell_height"
    launcher:widthGap="@dimen/workspace_width_gap"
    launcher:heightGap="@dimen/workspace_height_gap"
    launcher:maxGap="@dimen/workspace_max_gap" />
```

里面就一个CellLayout，也是一个自定义布局，那么我们就可以猜到了，既然可以存放图标，那么这个自定义的布局很有可能是继承自ViewGroup或者是其子类，实际上，CellLayout确实是继承自ViewGroup。在CellLayout里面，只放了一个子View，那就是ShortcutAndWidgetContainer。从名字也可以看出来，ShortcutAndWidgetContainer这个类就是用来存放**快捷图标**和**Widget小部件**的，那么里面放的是什么对象呢？

在桌面上的图标，使用的是BubbleTextView对象，这个对象在TextView的基础之上，添加了一些特效，比如你长按移动图标的时候，图标位置会出现一个背景\(不同版本的效果不同\)，所以我们找到BubbleTextView对象的点击事件，就可以找到Launcher如何开启一个App了。

除了在桌面上有图标之外，在程序列表中点击图标，也可以开启对应的程序。这里的图标使用的不是BubbleTextView对象，而是PagedViewIcon对象，我们如果找到它的点击事件，就也可以找到Launcher如何开启一个App。

其实说这么多，和今天的主题隔着十万八千里，上面这些东西，你有兴趣就看，没兴趣就直接跳过，不影响本章内容的阅读。

BubbleTextView的点击事件在哪里呢？在Launcher.onClick\(View v\)里面。

```text
/**
 * Launches the intent referred by the clicked shortcut
 */
public void onClick(View v) {

      ...ignore some code...

     Object tag = v.getTag();
    if (tag instanceof ShortcutInfo) {
        // Open shortcut
        final Intent intent = ((ShortcutInfo) tag).intent;
        int[] pos = new int[2];
        v.getLocationOnScreen(pos);
        intent.setSourceBounds(new Rect(pos[0], pos[1],
                pos[0] + v.getWidth(), pos[1] + v.getHeight()));
        //开始开启Activity
        boolean success = startActivitySafely(v, intent, tag);

        if (success && v instanceof BubbleTextView) {
            mWaitingForResume = (BubbleTextView) v;
            mWaitingForResume.setStayPressed(true);
        }
    } else if (tag instanceof FolderInfo) {
        //如果点击的是图标文件夹，就打开图标文件夹
        if (v instanceof FolderIcon) {
            FolderIcon fi = (FolderIcon) v;
            handleFolderClick(fi);
        }
    } else if (v == mAllAppsButton) {
    ...ignore some code...
    }
}
```

从上面的代码我们可以看到，在桌面上点击快捷图标的时候，会调用

```text
startActivitySafely(v, intent, tag);
```

那么从程序列表界面，点击图标的时候会发生什么呢？实际上，程序列表界面使用的是AppsCustomizePagedView对象，所以我在这个类里面找到了onClick\(View v\)。

com.android.launcher2.AppsCustomizePagedView.java

```text
/**
 * The Apps/Customize page that displays all the applications, widgets, and shortcuts.
 */
public class AppsCustomizePagedView extends PagedViewWithDraggableItems implements
        View.OnClickListener, View.OnKeyListener, DragSource,
        PagedViewIcon.PressedCallback, PagedViewWidget.ShortPressListener,
        LauncherTransitionable {

     @Override
    public void onClick(View v) {

            ...ignore some code...

        if (v instanceof PagedViewIcon) {
            mLauncher.updateWallpaperVisibility(true);
            mLauncher.startActivitySafely(v, appInfo.intent, appInfo);
        } else if (v instanceof PagedViewWidget) {
                   ...ignore some code..
         }
     }       
   }
```

可以看到，调用的是

```text
mLauncher.startActivitySafely(v, appInfo.intent, appInfo);
```

和上面一样！这叫什么？这叫殊途同归！

所以咱们现在又明白了一件事情：不管从哪里点击图标，调用的都是Launcher.startActivitySafely\(\)。

下面我们就可以一步步的来看一下Launcher.startActivitySafely\(\)到底做了什么事情。

```text
 boolean startActivitySafely(View v, Intent intent, Object tag) {
        boolean success = false;
        try {
            success = startActivity(v, intent, tag);
        } catch (ActivityNotFoundException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
        }
        return success;
    }
```

调用了startActivity\(v, intent, tag\)

```text
 boolean startActivity(View v, Intent intent, Object tag) {

        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        try {
            boolean useLaunchAnimation = (v != null) &&
                    !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);

            if (useLaunchAnimation) {
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent, opts.toBundle());
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(),
                            opts.toBundle());
                }
            } else {
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent);
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(), null);
                }
            }
            return true;
        } catch (SecurityException e) {
        ...
        }
        return false;
    }
```

这里会调用Activity.startActivity\(intent, opts.toBundle\(\)\)，这就是我们经常用到的Activity.startActivity\(Intent\)的重载函数。而且由于设置了

```text
 intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
```

所以这个Activity会添加到一个新的Task栈中，而且，startActivity\(\)调用的其实是startActivityForResult\(\)这个方法。

```text
@Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```

我们现在明确了，Launcher中开启一个App，其实和我们在Activity中直接startActivity\(\)基本一样，都是调用了Activity.startActivityForResult\(\)。

### Instrumentation是什么？和ActivityThread是什么关系？

每个Activity都持有Instrumentation对象的一个引用，但是整个进程只会存在一个Instrumentation对象。

当startActivityForResult\(\)调用之后，实际上还是调用了mInstrumentation.execStartActivity\(\)

```text
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            ...ignore some code...    
        } else {
            //在startActivityFromChild()内部实际还是调用的mInstrumentation.execStartActivity()
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
         ...ignore some code...    
    }
```

下面是mInstrumentation.execStartActivity\(\)的实现。

```text
 public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
              ...ignore some code...
      try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
        }
        return null;
    }
```

所以当我们在程序中调用startActivity\(\)的时候，实际上调用的是Instrumentation的相关的方法。

Instrumentation意为“仪器”，我们先看一下这个类里面包含哪些方法吧

![](http://i11.tietuku.com/15eb948d37d9d4b3.png)

我们可以看到，这个类里面的方法大多数和Application和Activity有关，是的，这个类就是完成对Application和Activity初始化和生命周期的工具类。比如说，单独挑一个callActivityOnCreate\(\)看看

```text
public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
```

对`activity.performCreate(icicle)`这一行代码熟悉吗？这一行里面就调用了Activity的入口函数onCreate\(\)，接着往下看

```text
final void performCreate(Bundle icicle) {
        onCreate(icicle);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }
```

onCreate在这里调用了吧。但是有一件事情必须说清楚，那就是这个Instrumentation类这么重要，为什么在开发的过程中，没有发现它的踪迹呢？

是的，Instrumentation这个类很重要，对Activity生命周期方法的调用根本就离不开他，他可以说是一个大管家，但是，这个大管家比较害羞，是一个女的，管内不管外，是老板娘~

那么你可能要问了，老板是谁呀？老板当然是大名鼎鼎的ActivityThread了！

ActivityThread所在线程就是UI线程。前面说过，App和AMS是通过Binder传递信息的，ActivityThread就是专门与AMS的外交工作的。

这里模拟一个打开Activity的场景：

* AMS说：“ActivityThread，你给我打开一个Activity！”
* ActivityThread就说：“没问题！”
* 然后ActivityThread转身和Instrumentation说：“老婆，AMS让打开一个Activity，我这里忙着呢，你快去帮我把这事办了把~”
* 于是，Instrumentation就去把事儿搞定了。

所以说，AMS是董事会，负责指挥和调度；ActivityThread是老板，虽然说家里的事自己说了算，但是需要听从AMS的指挥；Instrumentation则是老板娘，负责家里的大事小事，但是一般不抛头露面，听一家之主ActivityThread的安排。

### 如何理解AMS和ActivityThread之间的Binder通信？

前面我们说到，在调用startActivity\(\)的时候，实际上调用的是

```text
mInstrumentation.execStartActivity()
```

但是到这里还没完呢！里面又调用了下面的方法

```text
ActivityManagerNative.getDefault()
                .startActivity
```

这里的ActivityManagerNative.getDefault返回的就是ActivityManagerService的远程接口，即ActivityManagerProxy。

怎么知道的呢？往下看

```text
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{

//从类声明上，我们可以看到ActivityManagerNative是Binder的一个子类，而且实现了IActivityManager接口
 static public IActivityManager getDefault() {
        return gDefault.get();
    }

 //通过单例模式获取一个IActivityManager对象，这个对象通过asInterface(b)获得
 private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
}


//最终返回的还是一个ActivityManagerProxy对象
static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

     //这里面的Binder类型的obj参数会作为ActivityManagerProxy的成员变量保存为mRemote成员变量，负责进行IPC通信
        return new ActivityManagerProxy(obj);
    }


}
```

再看ActivityManagerProxy.startActivity\(\)，在这里面做的事情就是IPC通信，利用Binder对象，调用transact\(\)，把所有需要的参数封装成Parcel对象，向AMS发送数据进行通信。

```text
 public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(callingPackage);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```

Binder本质上只是一种底层通信方式，和具体服务没有关系。为了提供具体服务，Server必须提供一套接口函数以便Client通过远程访问使用各种服务。这时通常采用Proxy设计模式：将接口函数定义在一个抽象类中，Server和Client都会以该抽象类为基类实现所有接口函数，所不同的是Server端是真正的功能实现，而Client端是对这些函数远程调用请求的包装。

为了更方便的说明客户端和服务器之间的Binder通信，下面以ActivityManagerServices和他在客户端的代理类ActivityManagerProxy为例。

ActivityManagerServices和ActivityManagerProxy都实现了同一个接口——IActivityManager。

```text
class ActivityManagerProxy implements IActivityManager{}

public final class ActivityManagerService extends ActivityManagerNative{}

public abstract class ActivityManagerNative extends Binder implements IActivityManager{}
```

虽然都实现了同一个接口，但是代理对象ActivityManagerProxy并不会对这些方法进行真正地实现，ActivityManagerProxy只是通过这种方式对方法的参数进行打包\(因为都实现了相同接口，所以可以保证同一个方法有相同的参数，即对要传输给服务器的数据进行打包\)，真正实现的是ActivityManagerService。

但是这个地方并不是直接由客户端传递给服务器，而是通过Binder驱动进行中转。可以把Binder驱动当做一个中转站，客户端调用ActivityManagerProxy接口里面的方法，把数据传送给Binder驱动，然后Binder驱动就会把这些东西转发给服务器的ActivityManagerServices，由ActivityManagerServices去真正的实施具体的操作。

但是Binder只能传递数据，并不知道是要调用ActivityManagerServices的哪个方法，所以在数据中会添加方法的唯一标识码，比如前面的startActivity\(\)方法：

```text
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();

        ...ignore some code...

        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```

上面的**START\_ACTIVITY\_TRANSACTION**就是方法标示，data是要传输给Binder驱动的数据，reply则接受操作的返回值。

这种通信模型可以表示为：

客户端：ActivityManagerProxy =====&gt;Binder驱动=====&gt; AMS：服务器

而且由于继承了同样的公共接口类，ActivityManagerProxy提供了与AMS一样的函数原型，使用户感觉不出Server是运行在本地还是远端，从而可以更加方便的调用这些系统服务。

但是，这里Binder通信是单方向的，即从App指向AMS，如果AMS想要通知App做一些事情，应该怎么办呢？

还是通过Binder通信，不过是换了另外一对，换成了ApplicationThread和ApplicationThreadProxy。

这种通信模型可以表示为：

客户端：ApplicationThread &lt;=====Binder驱动&lt;===== ApplicationThreadProxy：服务器

他们也都实现了相同的接口IApplicationThread

```text
  private class ApplicationThread extends ApplicationThreadNative {}

  public abstract class ApplicationThreadNative extends Binder implements IApplicationThread{}

  class ApplicationThreadProxy implements IApplicationThread {}
```

剩下的就不必多说了，和前面是一样。

### AMS接收到客户端的请求之后，会如何开启一个Activity？

至此，点击桌面图标调用startActivity\(\)，终于把数据和要开启Activity的请求发送到了AMS了。说了这么多，其实这些都在一瞬间完成了，下面咱们研究下AMS到底做了什么。

由于下面整个过程很繁琐，所以不会深入源码介绍每个细节。

AMS收到startActivity的请求之后，会按照如下的方法链进行调用，

调用startActivity\(\)

```text
@Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }
```

调用startActivityAsUser\(\)

```text
 @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {

             ...ignore some code...

        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, userId, null, null);
    }
```

在这里又出现了一个新对象ActivityStackSupervisor，通过这个类可以实现对ActivityStack的部分操作。

```text
  final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, int userId, IActivityContainer iContainer, TaskRecord inTask) {

            ...ignore some code...

              int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options,
                    componentSpecified, null, container, inTask);

            ...ignore some code...

            }
```

继续调用startActivityLocked\(\)

```text
 final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean componentSpecified, ActivityRecord[] outActivity, ActivityContainer container,
            TaskRecord inTask) {

              err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
              startFlags, true, options, inTask);
        if (err < 0) {
            notifyActivityDrawnForKeyguard();
        }
        return err;
    }
```

调用startActivityUncheckedLocked\(\),此时要启动的Activity已经通过检验，被认为是一个正当的启动请求。

终于，在这里调用到了ActivityStack的startActivityLocked\(ActivityRecord r, boolean newTask,boolean doResume, boolean keepCurTransition, Bundle options\)。

ActivityRecord代表的就是要开启的Activity对象，里面封装了很多信息，比如所在的ActivityTask等，如果这是首次打开应用，那么这个Activity会被放到ActivityTask的栈顶，

```text
final int startActivityUncheckedLocked(ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {

            ...ignore some code...

            targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);

            ...ignore some code...

             return ActivityManager.START_SUCCESS;
            }
```

调用的是ActivityStack.startActivityLocked\(\)

```text
 final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {

        //ActivityRecord中存储的TaskRecord信息
        TaskRecord rTask = r.task;

         ...ignore some code...

        //如果不是在新的ActivityTask(也就是TaskRecord)中的话，就找出要运行在的TaskRecord对象
     TaskRecord task = null;
        if (!newTask) {
            boolean startIt = true;
            for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
                task = mTaskHistory.get(taskNdx);
                if (task.getTopActivity() == null) {
                    // task中的所有Activity都结束了
                    continue;
                }
                if (task == r.task) {
                    // 找到了
                    if (!startIt) {
                        task.addActivityToTop(r);
                        r.putInHistory();
                        mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                                r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                                (r.info.flags & ActivityInfo.FLAG_SHOW_ON_LOCK_SCREEN) != 0,
                                r.userId, r.info.configChanges, task.voiceSession != null,
                                r.mLaunchTaskBehind);
                        if (VALIDATE_TOKENS) {
                            validateAppTokensLocked();
                        }
                        ActivityOptions.abort(options);
                        return;
                    }
                    break;
                } else if (task.numFullscreen > 0) {
                    startIt = false;
                }
            }
        }

      ...ignore some code...

        // Place a new activity at top of stack, so it is next to interact
        // with the user.
        task = r.task;
        task.addActivityToTop(r);
        task.setFrontOfTask();

        ...ignore some code...

         if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
```

从ActivityStackSupervisor到ActivityStack，又回到了ActivityStackSupervisor，这到底是在折腾什么！

咱们再一起看下`StackSupervisor.resumeTopActivitiesLocked(this, r, options)`。

```text
 boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = getFocusedStack();
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

          ...ignore some code...

        return result;
    }
```

又调回ActivityStack去了。`ActivityStack.resumeTopActivityLocked()`。

```text
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            inResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            inResumeTopActivity = false;
        }
        return result;
    }
```

咱们坚持住，看一下`ActivityStack.resumeTopActivityInnerLocked()`到底进行了什么操作

```text
  final boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {

          ...ignore some code...
        //找出还没结束的首个ActivityRecord
       ActivityRecord next = topRunningActivityLocked(null);

      //如果一个没结束的Activity都没有，就开启Launcher程序
      if (next == null) {
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: No more activities go home");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            // Only resume home if on home display
            final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                    HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
            return isOnHomeDisplay() &&
                    mStackSupervisor.resumeHomeStackTask(returnTaskType, prev);
        }

        //先需要暂停当前的Activity。因为我们是在Lancher中启动mainActivity，所以当前mResumedActivity！=null，调用startPausingLocked()使得Launcher进入Pausing状态
          if (mResumedActivity != null) {
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Pausing " + mResumedActivity);
        }

  }
```

在这个方法里，prev.app为记录启动Lancher进程的ProcessRecord，prev.app.thread为Lancher进程的远程调用接口IApplicationThead，所以可以调用prev.app.thread.schedulePauseActivity，到Lancher进程中暂停指定Activity。

```text
 final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
        if (mPausingActivity != null) {
            completePauseLocked(false);
        }

       ...ignore some code...    

        if (prev.app != null && prev.app.thread != null) 
          try {
                mService.updateUsageStats(prev, false);
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }

      ...ignore some code...  

 }
```

在Lancher进程中消息传递，调用ActivityThread.handlePauseActivity\(\)，最终调用ActivityThread.performPauseActivity\(\)暂停指定Activity。接着通过Binder通信，通知AMS已经完成暂停的操作。

```text
ActivityManagerNative.getDefault().activityPaused(token).
```

上面这些调用过程非常复杂，源码中各种条件判断让人眼花缭乱，所以说如果你没记住也没关系，你只要记住这个流程，理解了Android在控制Activity生命周期时是如何操作，以及是通过哪几个关键的类进行操作的就可以了，以后遇到相关的问题之道从哪块下手即可。

最后来一张高清无码大图，方便大家记忆：

[请戳这里\(图片3.3M，请用电脑观看\)](http://i11.tietuku.com/0582844414810f38.png)

### 彩蛋

#### 不要使用 startActivityForResult\(intent,RESULT\_OK\)

这是因为startActivity\(\)是这样实现的

```text
public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```

而

```text
 public static final int RESULT_OK  = -1;
```

所以

```text
startActivityForResult(intent,RESULT_OK) = startActivity()
```

你不可能从onActivityResult\(\)里面收到任何回调。而这个问题是相当难以被发现的，就是因为这个坑，我工作一年多来第一次加班到9点 \(ˇˍˇ）

#### 一个App的程序入口到底是什么？

是ActivityThread.main\(\)。

#### 整个App的主线程的消息循环是在哪里创建的？

是在ActivityThread初始化的时候，就已经创建消息循环了，所以在主线程里面创建Handler不需要指定Looper，而如果在其他线程使用Handler，则需要单独使用Looper.prepare\(\)和Looper.loop\(\)创建消息循环。

```text
 public static void main(String[] args) {

          ...ignore some code...    

       Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

          ...ignore some code...    

 }
```

### Application是在什么时候创建的？onCreate\(\)什么时候调用的？

也是在ActivityThread.main\(\)的时候，再具体点呢，就是在thread.attach\(false\)的时候。

我们先看一下ActivityThread.attach\(\)

```text
private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        //普通App进这里
        if (!system) {

            ...ignore some code...    

            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
           } else {
             //这个分支在SystemServer加载的时候会进入，通过调用
             // private void createSystemContext() {
             //    ActivityThread activityThread = ActivityThread.systemMain()；
             //} 

             // public static ActivityThread systemMain() {
        //        if (!ActivityManager.isHighEndGfx()) {
        //            HardwareRenderer.disable(true);
        //        } else {
        //            HardwareRenderer.enableForegroundTrimming();
        //        }
        //        ActivityThread thread = new ActivityThread();
        //        thread.attach(true);
        //        return thread;
        //    }       
           }
    }
```

这里需要关注的就是mgr.attachApplication\(mAppThread\)，这个就会通过Binder调用到AMS里面对应的方法

```text
@Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

然后就是

```text
 private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {


             thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat, getCommonServicesLocked(),
                    mCoreSettingsObserver.getCoreSettingsLocked());


            }
```

thread是IApplicationThread，实际上就是ApplicationThread在服务端的代理类ApplicationThreadProxy，然后又通过IPC就会调用到ApplicationThread的对应方法

```text
private class ApplicationThread extends ApplicationThreadNative {

  public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
                Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
                Bundle coreSettings) {

                 ...ignore some code...    

             AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableOpenGlTrace = enableOpenGlTrace;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            sendMessage(H.BIND_APPLICATION, data);

           }

}
```

我们需要关注的其实就是最后的sendMessage\(\)，里面有函数的编号H.BIND\_APPLICATION，然后这个Messge会被H这个Handler处理

```text
 private class H extends Handler {

       ...ignore some code... 

      public static final int BIND_APPLICATION        = 110;

       ...ignore some code... 

      public void handleMessage(Message msg) {
          switch (msg.what) {
         ...ignore some code... 
          case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
         ...ignore some code... 
         }
 }
```

最后就在下面这个方法中，完成了实例化，拨那个企鹅通过mInstrumentation.callApplicationOnCreate实现了onCreate\(\)的调用。

```text
 private void handleBindApplication(AppBindData data) {

 try {

           ...ignore some code... 

            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;

           ...ignore some code... 

            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
            }
            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
 }
```

data.info是一个LoadeApk对象。 LoadeApk.data.info.makeApplication\(\)

```text
 public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                initializeJavaContextClassLoader();
            }
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

    //传进来的是null，所以这里不会执行，onCreate在上一层执行
        if (instrumentation != null) {
            try {
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {

            }
        }
         ...ignore some code... 

       }

        return app;
    }
```

所以最后还是通过Instrumentation.makeApplication\(\)实例化的，这个老板娘真的很厉害呀！

```text
static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
        app.attach(context);
        return app;
    }
```

而且通过反射拿到Application对象之后，直接调用attach\(\)，所以attach\(\)调用是在onCreate\(\)之前的。

## 参考文章

下面的这些文章都是这方面比较精品的，希望你抽出时间研究，这可能需要花费很长时间，但是如果你想进阶为中高级开发者，这一步是必须的。

再次感谢下面这些文章的作者的分享精神。

### Binder

* [Android Bander设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)

### zygote

* [Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)
* [Android 之 zygote 与进程创建](http://blog.csdn.net/xieqibao/article/details/6581975)
* [Zygote浅谈](http://www.th7.cn/Program/Android/201404/187670.shtml)

### ActivityThread、Instrumentation、AMS

* [Android Activity.startActivity流程简介](http://blog.csdn.net/myarrow/article/details/14224273)
* [Android应用程序进程启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6747696#comments)
* [框架层理解Activity生命周期\(APP启动过程\)](http://laokaddk.blog.51cto.com/368606/1206840)
* [Android应用程序窗口设计框架介绍](http://blog.csdn.net/yangwen123/article/details/35987609)
* [ActivityManagerService分析一：AMS的启动](http://www.xuebuyuan.com/2172927.html)
* [Android应用程序窗口设计框架介绍](http://blog.csdn.net/yangwen123/article/details/35987609)

### Launcher

* [Android 4.0 Launcher源码分析系列\(一\)](http://mobile.51cto.com/hot-312129.htm)
* [Android Launcher分析和修改9——Launcher启动APP流程](http://www.cnblogs.com/mythou/p/3187881.html)

