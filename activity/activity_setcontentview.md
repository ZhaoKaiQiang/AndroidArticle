# setContentView\(\)之后是如何实现界面显示的？

你有没有想过，当我们在Activity.onCreate\(\)中调用setContentView\(\)设置显示界面的时候，背后到底发生了哪些事情呢？

```text
@Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
```

这一篇文章就是介绍这个过程的。

在Activity中存在着三个重载函数，不过最常用的是第一个，其他方法类似，下面我们重点看一下第一个重载函数的实现过程。

framework/base/core/java/android/app/Activity.java

```text
    public void setContentView(int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }

    public void setContentView(View view) {
        getWindow().setContentView(view);
        initWindowDecorActionBar();
    }

    public void setContentView(View view, ViewGroup.LayoutParams params) {
        getWindow().setContentView(view, params);
        initWindowDecorActionBar();
    }
```

首先分析`getWindow().setContentView()`到底做了什么。

getWindow\(\)返回的是一个Window对象，而Window是一个抽象类，这里的实际类型为Window的子类PhoneWindow。

framework/base/core/java/android/app/Activity.java

```text
private Window mWindow;

public Window getWindow() {
        return mWindow;
    }
```

在`PhoneWindow.setContentView()`中做了下面的事情

frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java

```text
@Override
    public void setContentView(int layoutResID) {

        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        //这里的Callback是Activity，isDestroyed()判断的也是Activity是否已经调用Destory()
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            //在Activity中，onContentChanged()是空方法
            cb.onContentChanged();
        }
    }
```

