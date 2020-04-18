# 从startActivity\(\)到UI对用户可见的过程中发生了什么事情？

我们调用`Activity.startActivity()`开启一个新的Activity时，AMS会协调整个系统实现资源的准备，最终开启一个Activity的具体操作，还是会调用`ActivityThread.handleLaunchActivity()`来完成。

```text
 private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {

        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        ...

        //Activity.attach()、Activity.setTheme()、Activity.onCreate()、Activity.onStart()、Activity.onRestoreInstanceState()、Activity.onPause()在这里被依次调用，具体的介绍会在Activity生命周期机制进行介绍
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            //正常情况下，Activity.onResume()会被调用，如果有需要的话，Activity.onRestart()、Activity.onStart()会在Activity.onResume()之前调用
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);

            ...

        } else {
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {
                // Ignore
            }
        }
    }
```

