# Activity在各种状态下的生命周期

Activity的生命周期虽然是基础知识点，在刚开始学习Android的时候就已经学习过。但是实际上，如果要考虑其他与生命周期相关的方法，情况还是比较复杂的，熟练的掌握Activity在常见情况下的各种生命周期变化，有助于我们写出稳健的代码。

## 正常情况下的Activity生命周期

下面这张图完整的显示了Activity的生命周期方法。

![](http://img.my.csdn.net/uploads/201212/15/1355546880_5050.png)

在正常情况下，总结起来分为以下几种情况：

1. 开启新的Activity A时，A会经历onCreate\(\)-&gt;onStart\(\)-&gt;onResume\(\)三个阶段，最后停留在Resumed状态
2. 当点击Back按键返回时，A会经历onPause\(\)-&gt;onStop\(\)-&gt;onDestory\(\)三个阶段，最后销毁
3. 当点击Home按键返回桌面时，A会经历onPause\(\)-&gt;onStop\(\)，若开启其他App造成内存紧张时，A会在未来的某个时间点onDestory\(\)
4. 当A从情况3的状态再次回到前台时，A会经历情况1的整个流程，恢复到Resumed状态
5. 当从A打开Activity B\(非Dialog主题\)时，A会经历onPause\(\)-&gt;onStop\(\)两个阶段，B会经历情况1下的整个流程
6. 当从B点击Back按钮回到A时，A会经历onRestart\(\)-&gt;onStart\(\)-&gt;onResume\(\)，再次进入Resumed状态，B会经历情况2的整个流程
7. 当A打开一个Dialog主题的Activity时，A会经历onPause\(\)，进入Paused状态。特别注意：在A中打开Dialog时，A的生命周期方法不会有任何反应，但是有其他回调事件，在下面的内容里会详细说明

另外需要特别注意的是，当从Activity A打开Activity B时，A和B的生命周期并非是顺序执行的，而是穿插执行，具体的执行顺序如下：

* A onPause\(\)
* B onCreate\(\)
* B onStart\(\)
* B onResume\(\)
* A onStop\(\)

了解这个知识点的意义在于：

* 如果A和B有共享数据，而且A在onStop\(\)中对共享数据进行修改的话，会覆盖B在生命周期函数中对同一个数据的修改，有可能会造成数据的污染
* 不要在A的onPause\(\)中进行过多的数据操作，这样会影响B的开启速度

特殊情况下，如果在onCreate\(\)中直接调用finish\(\)，那么整个生命周期只会调用onCreate\(\)-&gt;onDestory\(\)，其他的生命周期方法不会调用。

## onSaveInstanceState\(\)、onRestoreInstanceState\(\)

我们都知道，为了能够在Activity被意外销毁的情况下，再次显示时可以恢复现场，Android本身提供了这样的机制，核心方法就是onSaveInstanceState\(\)、onRestoreInstanceState\(\)。

### onSaveInstanceState\(\)在什么情况下会被调用呢？

经过测试，onSaveInstanceState\(\)在以下几种情况下会被调用：

1. 点击Home回到桌面
2. 打开Dialog主题的Activity
3. 打开普通的Activity
4. 锁屏
5. 切换屏幕方向导致Activity销毁重建
6. 查看最近任务列表

从上面这六种情况，我们可以总结出onSaveInstanceState\(\)的调用时机：当系统认为当前Activity有被意外杀死的风险时，会调用onSaveInstanceState\(\)。

### onRestoreInstanceState\(\)在什么情况下可以被调用呢？

要注意的是，虽然onSaveInstanceState\(\)在以上情况下都会被调用，但是onRestoreInstanceState\(\)并不是每次都会调用，即onSaveInstanceState\(\)与onRestoreInstanceState\(\)并不是配对调用的。

onRestoreInstanceState\(\)在以下情况下会被调用：

1. 切换屏幕方向导致Activity销毁重建
2. 切换到后台被系统销毁，再次进入时

从以上这两种情况我们可以发现，只有当Activity被系统销毁，而且有恢复现场的需求时，onRestoreInstanceState\(\)才会被调用。

### onSaveInstanceState\(\)、onRestoreInstanceState\(\)与Activity的生命周期函数之间有什么关系？

onSaveInstanceState\(\)一定会在onStop\(\)之前被调用，但是与onPause\(\)之间并不存在固定的顺序。

onRestoreInstanceState\(\)一定在onStart\(\)之后，onResume\(\)之前被执行，这个是有严格的顺序的。

当存在现场恢复的需求时，onRestoreInstanceState\(\)和onCreate\(\)参数中的Bundle对象都含有onSaveInstanceState\(\)中保存的数据，因此可以据此进行现场恢复。

当onRestoreInstanceState\(\)被调用时，Bundle对象一定不为空，onCreate\(\)只有在现场恢复的场景下才不为null。

### onSaveInstanceState\(\)里面默认会保存哪些数据？

这一点从源码的角度很容易得到答案。

```text
protected void onSaveInstanceState(Bundle outState) {
    //保存View Tree中的各种状态
    outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
    Parcelable p = mFragments.saveAllState();
    if (p != null) {
        //保存Fragment的状态
        outState.putParcelable(FRAGMENTS_TAG, p);
    }
    getApplication().dispatchActivitySaveInstanceState(this, outState);
}
```

对于界面的保存工作被分发到了Window，View的子类都重写了这个方法，来保存自己的逻辑，具体的流程可以参考**Activity与Window是什么关系？**。

还有一个需要注意的问题是，系统只会默认保存有id的View的状态。其他额外信息通过Bundle对象可以存储。

## onWindowFocusChanged

onWindowFocusChanged\(\)不属于生命周期方法，但是容易与生命周期方法混淆，所以这里一并介绍。

当当前Activity绑定的Windowon焦点状态发生变化时，WindowFocusChanged\(\)会被调用，在以下情况下会触发onWindowFocusChanged\(\):

* 锁屏时，此时失去焦点
* 解屏时，此时得到焦点
* 显示Dialog时，此时失去焦点
* 消失Dialog时，此时得到焦点
* 显示新的Activity\(包括Dialog主题\)时，失去焦点
* 从其他Activity\(包括Dialog主题\)回来时，得到焦点
* 首次进入时，得到焦点，此时View Tree已正常初始化，可以获取到View的宽高属性
* Back按键返回时，失去焦点
* 下拉通知栏时，失去焦点
* 上拉通知栏时，得到焦点

从上面几种情况下所引发的的焦点变化来看，我们可以总结出一些规律，那就是如果当前Activity可以与用户进行交互，比如触摸、按钮点击等，那么当前Activity一定是获取焦点的，换句话说，焦点所在的Window可以处理用户交互事件。

一般来说，焦点的变化会便随着Activity的一些生命周期方法的调用，但是存在特殊情况，Dialog和下拉通知栏可以引起焦点的变化，却不会引起生命周期方法的调用，这一点需要特别注意。

