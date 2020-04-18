# Touch事件分发机制

## Touch事件的整体传递轨迹

硬件传感器-&gt;System Server-&gt;Activity-&gt;Window-&gt;DecorView-&gt;内部ViewGroup和View

## 参与Touch事件传递的角色及对应方法

### Activity

Activity中与Touch事件传递相关的事件如下：

* public boolean dispatchTouchEvent\(MotionEvent ev\)
* public boolean onTouchEvent\(MotionEvent event\) 
* public void onUserInteraction\(\)

dispatchTouchEvent\(\)可以认为是Touch事件在App中传递的第一站，从这里开始，Touch事件就开始进入到传递的轨道上了。该方法的返回值取决于Activity布局内部所有的View及ViewGroup是否处理Touch事件以及自身的`onTouchEvent()`的返回值，这一点可以在源码中得到解答.

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

当Activity布局处理Touch事件时，返回true，否则返回值就依赖于`Activity.onTouchEvent()`。

另外还有一个不太常见的方法，就是`onUserInteraction`，当Touch事件是ACTION\_DOWN时，便会触发一次。其实`onUserInteraction`不只是在Touch事件中被调用，Back按钮点击，轨迹球滚动和Touch事件都会触发这个回调方法。

当我们不想Touch事件在当前Activity中传递时应该怎么做呢？很简单，我们只需要重写dispatchTouchEvent\(\)，不去调用`super.dispatchTouchEvent()`即可阻止Touch事件在当前Activity的传递，如下，这样这个Activity就再也处理不了Touch事件啦！他被我给废了！

```text
public class NoTouchActivity extends Activity {

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        return false;
    }
```

当然，一般情况下我们不需要对Activity的这些方法做任何处理，这样伤害太大啦~

### Window（PhoneWindow）

* public boolean superDispatchTouchEvent\(MotionEvent ev\)

### View

* dispatchTouchEvent\(\)
* onTouchEvent\(\)

### ViewGroup

* dispatchTouchEvent\(\)
* onInterceptTouchEvent\(\)
* onTouchEvent\(\)

