# ActivityManagerService是什么？有什么作用？

ActivityManagerService，以下简称AMS，服务端对象，负责系统中所有Activity、Service的生命周期以及BroadcastReceiver和ContentProvider的管理。

在SystemServer进程开启的时候，就会初始化AMS，从下面的代码中可以看到

```text
public final class SystemServer {

  //zygote的主入口
    public static void main(String[] args) {
        new SystemServer().run();
    }

    public SystemServer() {
        // Check for factory test mode.
        mFactoryTestMode = FactoryTest.getMode();
    }

    private void run() {

      ...ignore some code...

      //加载本地系统服务库，并进行初始化 
        System.loadLibrary("android_servers");
        nativeInit();

        createSystemContext();

        //初始化SystemServiceManager对象，下面的系统服务开启都需要调用SystemServiceManager.startService(Class<T>)
        //这个方法通过反射来启动对应的服务
        mSystemServiceManager = new SystemServiceManager(mSystemContext);

        //开启服务
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            throw ex;
        }

        ...ignore some code...

    }

  //初始化系统上下文对象mSystemContext，mSystemContext实际上是一个ContextImpl对象
  //调用ActivityThread.systemMain()的时候，会调用ActivityThread.attach(true)
  private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
    }

  //在这里开启了几个核心的服务，因为这些服务之间相互依赖，所以都放在了这个方法里面。
  private void startBootstrapServices() {

    ...ignore some code...

    //初始化AMS
    mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);

        //初始化PowerManagerService，因为其他服务需要依赖这个Service，因此需要尽快的初始化
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

        // 现在电源管理已经开启，ActivityManagerService负责电源管理功能
        mActivityManagerService.initPowerManagement();

      //初始化PackageManagerService
      mPackageManagerService = PackageManagerService.main(mSystemContext, mInstaller,
           mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);

  ...ignore some code...

  }

}
```

经过上面这些步骤，我们的AMS已经创建好了，并且完成了成员变量初始化。在这之后，会开启系统的Launcher程序，完成系统界面的加载与显示。

你是否会好奇，为什么说AMS是服务端对象？下面简单介绍下Android系统里面的服务器和客户端的概念。

服务器客户端\(以下简称CS架构\)的概念不仅仅存在于Web开发中，在Android的框架设计中，使用的也是CS架构。服务器端指的就是所有App共用的系统服务，比如我们这里提到的AMS、PMS、WMS等，这些基础的系统服务是被所有的App公用的，当某个App想实现某个操作的时候，要告诉这些系统服务。

比如想打开一个App，那么知道了包名和MainActivity之后就可以：

```text
Intent intent = new Intent(Intent.ACTION_MAIN);  
intent.addCategory(Intent.CATEGORY_LAUNCHER);              
ComponentName cn = new ComponentName(packageName, className);              
intent.setComponent(cn);  
startActivity(intent);
```

但是，我们的App通过startActivity\(\)并不能直接打开另外一个App，这个请求会通过一系列的调用，最后还是告诉AMS说：“我要打开某个App，我知道他的住址和名字，你帮我打开吧！”如果目标App没有启动过的话，AMS会通知zygote进程首先fork一个新进程来容纳目标App。这就像是浏览器想要打开一个超链接一样，浏览器把网页地址发送给服务器，最终是服务器把需要的资源文件发送给客户端完成显示的。

了解了Android的CS架构之后，我们还需要了解一件事情，那就是我们的App和AMS\(SystemServer进程\)还有zygote进程分属于三个独立的进程，他们之间如何通信呢？

App与AMS通过Binder进行IPC通信，AMS\(SystemServer进程\)与zygote通过Socket进行IPC通信。

在Android系统中，任何一个Activity的启动都是由AMS和应用程序进程相互配合来完成的。AMS服务统一调度系统中所有进程的Activity的生命周期，而每个Activity的具体操作过程则由其所属的进程来完成。

AMS除了可以管理Activity之外，Service、BroadcastReceiver和ContentProvider也都和AMS有关联，从这里也可以看出AMS的重要性。

这样说可能还是比较抽象，在**Activity**机制一章中介绍了AMS与ActivityThread如何一起配合，完成Activity生命周期的调度。

