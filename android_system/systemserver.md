# SystemServer是什么？

首先要了解的是SystemServer本质上是一个进程，而且是由zygote进程fork出来的。

知道了SystemServer的本质，我们对它就不算太陌生了，这个进程是Android里两大重要进程之一——另外一个进程就是zygote。

为什么说SystemServer非常重要呢？因为系统里面重要的服务都是在这个进程里面开启的，比如 ActivityManagerService、PackageManagerService、WindowManagerService等等。

那么这些系统服务是怎么开启起来的呢？

在zygote开启的时候，会调用ZygoteInit.main\(\)进行初始化

```text
public static void main(String argv[]) {

   ...ignore some code...

  //在加载首个zygote的时候，会传入初始化参数，使得startSystemServer = true
   boolean startSystemServer = false;
   for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            ...ignore some code...

    //开始fork我们的SystemServer进程
  if (startSystemServer) {
        startSystemServer(abiList, socketName);
    }

   ...ignore some code...

}
```

我们看下startSystemServer\(\)做了些什么

```text
    /**SystemServer确实是被fork出来的
     * Prepare the arguments and fork for the system server process.
     */
    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {

         ...ignore some code...

        //ZygoteInit.main(String argv[])里面的argv就是通过这种方式传递进来的
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };

        int pid;
        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

    //确实是fuck出来的吧，我没骗你吧~不对，是fork出来的 -_-|||
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
```

