# 总结

在前面几篇文章中，我们比较深入的分析了Handler机制的整个流程和一些细节，下面我们一起总结一下Handler中比较重要的部分：

* Handler机制主要由Handler、MessageQueue、Message、Looper组成
* Handler在整个结构中扮演的角色是总指挥，我们主要通过Handler的一些方法来完成Message的发送和处理
* Message是消息载体，在其中包含了我们需要传递的数据，其内部维持着一个最大容量为50\(不同版本之间可能存在差异\)的静态缓存池，数据结构类型为单链表结构，由于`Message.obtain()`、`Message.recycleUnchecked()`存在多线程访问静态变量的场景，因此需要进行线程同步操作
* 每个MessageQueue内部都持有一个Message对象mMessages，mMessage是所有还未处理的Message的链表头。MessageQueue主要为链表提供了插入、取出、查询、删除功能，由于Handler支持延时任务，因此在进行插入操作时，会根据`Message.when`将Message插入到合适的位置。
* Handler机制属于很典型的生产者消费者模型，调用Handler的线程就是生产者，生产要执行的Message；Looper的绑定线程就是消费者，负责消化（执行）任务。Message通过MessageQueue由生产者传递给消费者，就形成了N个生产者，1个消费者的模型。
* 线程的等待和唤醒操作由MessageQueue类中的`nativePollOnce`、`nativeWake`负责完成，这一部分的实现是在Native层，这里不过多深究
* UI线程拥有自己的Looper消息循环，并且在程序退出之前，会一直处于`Looper.loop()`方法体内，ActivityThread与AMS之间的Binder通信需要通过名为H的Handler作为中转，当我们在UI线程中修改View属性造成界面的重绘、触发触摸点击等操作时，最终的实现方式也是发送Message到主线程的MessageQueue，然后由Main Looper取出进行处理。

