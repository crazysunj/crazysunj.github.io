---
title: Android系统启动过程笔记
toc: true
date: 2018-04-12 11:48:23
tags: [笔记,系统]
---

Android设备上电后，首先会从处理器片上ROM的启动引导代码开始执行，片上ROM会寻找Bootloader代码，并加载到内存

Bootloader开始执行，首先负责完成硬件的初始化，然后找到Linux内核代码，并加载到内存

Linux内核开始启动，初始化各种软硬件环境，加载驱动程序，挂载根文件系统，解析init.rc脚本，启动两个比较重要的进程：servicemanager和zygote
<!--  more-->
```
...
service servicemanager /system/bin/servicemanager
    class core
    user system
    group system
    critical
    onrestart restart zygote
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart drm
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
...
```

同时启动servicemanager管理的几个native service，如media，surfaceflinger，drm

zygote启动java层的ZygoteInit

```
 int main(int argc, const char*const argv[]) {
    ...
    /*runtime就是dalvik虚拟机实例，启动Java层应用时，
     *会fork 一个子进程，复制虚拟机，许多书上将runtime看作一个进程，
     *然后再启动zygote进程，个人觉得这是错误的
     */
    AppRuntime runtime;
    ...
    while (i < argc) {
        const char*arg = argv[i++];
        if (!parentDir) {
            parentDir = arg;
            /*init.rc启动app_main会设置参数--zygote*/
        } else if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = "zygote"; //进程的名字
            /*init.rc启动app_main会设置参数--start-system-server，
             *表示需启动systemserver
             */
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
            /*启动应用时会使用--application参数*/
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
            /*--nice-name=参数表示要设置的进程名字*/
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName = arg + 12;
        } else {
            className = arg;
            break;
        }
    }
    /*设置进程名*/
    if (niceName && *niceName){
        setArgv0(argv0, niceName);
        set_process_name(niceName);
    }
    /*设置虚拟机运行环境的父目录*/
    runtime.mParentDir = parentDir;
    if (zygote) {
        /*虚拟机里启动com.android.internal.os.ZygoteInit，
         *并传递参数start-system-server
         */
        runtime.start("com.android.internal.os.ZygoteInit",
                startSystemServer ? "start-system-server" : "");
    } else if (className) {
        /*若不是zygote，则启动的第一个类是com.android.internal.os.RuntimeInit，
         *RumtimeInit初始化后会启动mClassName
         */
        runtime.mClassName = className;
        runtime.mArgC = argc - i;
        runtime.mArgV = argv + i;
        runtime.start("com.android.internal.os.RuntimeInit",
                application ? "application" : "tool");
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
    ...
}
```

ZygoteInit启动SystemServer

```
public static void main(String argv[]) {
    try {
        ...
        //在某个描述符上监听连接请求，
        //其它Android空间的程序的启动都是通过连接zygote才孵化出来的
        registerZygoteSocket();
        ...
        if (argv[1].equals("start-system-server")) {
            //启动SystemServer
            startSystemServer();
        } else if (!argv[1].equals("")) {
            throw new RuntimeException(argv[0] + USAGE_STRING);
        }
        ...
        /*ZYGOTE_FORK_MODE默认为false，如果为true的话，每收到一个连接请求，
         *就会建立一个新进程，然后再运行连接请求所要求执行的命令，此时会建立另一个新进程
         */
        if (ZYGOTE_FORK_MODE) {
            runForkMode();
        } else {
            //使用Select poll的方式来建立新进程，收到连接请求后，也会建立进程启动某个程序
            runSelectLoopMode();
        }
        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}
```

```
private static boolean startSystemServer() {
    /* Hardcoded command line to start the system server */
	//启动SystemServer使用的参数
    String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,3001,3002,3003,3004,3006,3007,3009",
            "--capabilities=130104352,130104352",
            "--runtime-init",
            "--nice-name=system_server",
            //注意：就是在这里设置要启动的SystemServer包名及类名，故此后续才能启动SystemServer
            "com.android.server.SystemServer",
    };
    ZygoteConnection.Arguments parsedArgs = null;
    int pid;
    try {
        /*将args参数传给ZygoteConnection进行转化，--形式的参数将全部被接收
         * 但是要启动的类的类名com.android.server.SystemServer会放在
         *ZygoteConnection.Arguments的remainingArgs里，后来调用handleSystemServerProcess时会用到
         */
        parsedArgs = new ZygoteConnection.Arguments(args);
        /*添加额外运行参数*/
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
        /*开启新进程*/
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
	    /*调用handleSystemServerProcess会执行ZygoteConnection.Arguments的remainingArgs参数
	    *所指定的类，即com.android.server.SystemServer\t
	    */
        handleSystemServerProcess(parsedArgs);
    }
}
```
ZygoteInit的startSystemServer会调用handleSystemServerProcess来真正启动systemserver

```
private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {
    ...
    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName);
    }
	//启动systemserver时invokeWith为null
    if (parsedArgs.invokeWith != null) {
        WrapperInit.execApplication(parsedArgs.invokeWith,
                parsedArgs.niceName, parsedArgs.targetSdkVersion,
                null, parsedArgs.remainingArgs);
    } else {
       /*
        * 启动systemserver时，parsedArgs.remainingArgs为com.android.server.SystemServer.
        */
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs);
    }
}
```

RuntimeInit.zygoteInit-> applicationInit

```
private static void applicationInit(int targetSdkVersion, String[] argv) {
    ...
    final Arguments args;
    try {
        //参数转换,系统启动时，argv里有一个参数是com.android.server.SystemServer
        args = new Arguments(argv);
    } catch (IllegalArgumentException ex) {
        Slog.e(TAG, ex.getMessage());
        // let the process exit
        return;
    }
    ...
	//终于在此启动了SystemServer
    invokeStaticMain(args.startClass, args.startArgs)
}
```

SystemServer的main方法中会调用init1方法，该方法是个native方法，jni对应android_server_SystemServer_init1，该函数又会调用system_init

```
extern "C"
status_t system_init() {
 	...
    sp<ProcessState> proc (ProcessState::self ());
    sp<IServiceManager> sm = defaultServiceManager();
    ALOGI("ServiceManager: %p\\n", sm.get());
    sp<GrimReaper> grim = new GrimReaper();
    sm -> asBinder()->linkToDeath(grim, grim.get(), 0);
    char propBuf[ PROPERTY_VALUE_MAX];
    property_get("system_init.startsurfaceflinger", propBuf, "1");
    if (strcmp(propBuf, "1") == 0) {
        // Start the SurfaceFlinger
        SurfaceFlinger::instantiate ();
    }
    property_get("system_init.startsensorservice", propBuf, "1");
    if (strcmp(propBuf, "1") == 0) {
        // Start the sensor service
        SensorService::instantiate ();
    }
	// And now start the Android runtime.  We have to do this bit
	// of nastiness because the Android runtime initialization requires
	// some of the core system services to already be started.
	// All other servers should just start the Android runtime at
	// the beginning of their processes's main(), before calling
	// the init function.
    ALOGI("System server: starting Android runtime.\\n");
    AndroidRuntime * runtime = AndroidRuntime::getRuntime ();
    ALOGI("System server: starting Android services.\\n");
    JNIEnv * env = runtime -> getJNIEnv();
    if (env == NULL) {
        return UNKNOWN_ERROR;
    }
    jclass clazz = env -> FindClass("com/android/server/SystemServer");
    if (clazz == NULL) {
        return UNKNOWN_ERROR;
    }
	//反过来调用Java里SystemServer的init2函数
    jmethodID methodId = env -> GetStaticMethodID(clazz, "init2", "()V");
    if (methodId == NULL) {
        return UNKNOWN_ERROR;
    }
    env -> CallStaticVoidMethod(clazz, methodId);
    ALOGI("System server: entering thread pool.\\n");
    ProcessState::self () -> startThreadPool();
    IPCThreadState::self () -> joinThreadPool();
    ALOGI("System server: exiting thread pool.\\n");
}
```
上面注释已经说明会反过来去调用SystemServer的init2方法，会开启新线程android.server.ServerThread，在新线程里会启动各种Java层的service并注册，但著名的AMS并非在其中，而是在线程ServerThread中调用ActivityManagerService.setSystemProcess

```
public static void setSystemProcess() {
    ...
    ActivityManagerService m = mSelf;
    ServiceManager.addService("activity", m, true);
    ServiceManager.addService("meminfo", new MemBinder(m));
    ServiceManager.addService("gfxinfo", new GraphicsBinder(m));
    ServiceManager.addService("dbinfo", new DbBinder(m));
    if (MONITOR_CPU_USAGE) {
        ServiceManager.addService("cpuinfo", new CpuBinder(m));
    }
    ServiceManager.addService("permission", new PermissionController(m));
    ApplicationInfo info =
            mSelf.mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS);
    mSystemThread.installSystemApplicationInfo(info);
    ...
}
```
mSelf为static，可以猜到ActivityManagerService是单例，且其它service(MemBinder、GraphicsBinder和CpuBinder等)都有AMS的引用

AMS注册后将启动系统界面

```
// android.server.ServerThread
ActivityManagerService.self().systemReady(new Runnable() {
    public void run() {
        Slog.i(TAG, "Making services ready");
        if (!headless) startSystemUi(contextF);
        ...
        mMainStack.resumeTopActivityLocked(null);
        ...
     }
}

static final void startSystemUi(Context context) {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName("com.android.systemui",
                    "com.android.systemui.SystemUIService"));
        //Slog.d(TAG, "Starting service: " + intent);
        context.startServiceAsUser(intent, UserHandle.OWNER);
}
```

启动com.android.systemui应用

mMainStack就是ActivityStack，调用其resumeTopActivityLocked方法

```
...
if (next == null) {
    // There are no more activities!  Let's just start up the
    // Launcher...
    if (mMainStack) {
        ActivityOptions.abort(options);
        return mService.startHomeActivityLocked(mCurrentUser);
    }
}
...
```
mService就是AMS，其中startHomeActivityLocked方法

```
 boolean startHomeActivityLocked(int userId) {
    ...
    Intent intent = new Intent(
            mTopAction,
            mTopData != null ? Uri.parse(mTopData) : null);
    intent.setComponent(mTopComponent);
	//这里便添加了Intent.CATEGORY_HOME，
	//所有的Home应用都会都带有该类型的Activity，只有这样才会被认为是Home应用
    if (mFactoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) {
        intent.addCategory(Intent.CATEGORY_HOME);
    }
    ActivityInfo aInfo =
            resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
    if (aInfo != null) {
        intent.setComponent(new ComponentName(
                aInfo.applicationInfo.packageName, aInfo.name));
        // Don't do this if the home app is currently being
        // instrumented.
        aInfo = new ActivityInfo(aInfo);
        aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
        ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                aInfo.applicationInfo.uid);
        if (app == null || app.instrumentationClass == null) {
            intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
            mMainStack.startActivityLocked(null, intent, null, aInfo,
                    null, null, 0, 0, 0, 0, null, false, null);
        }
    }
}
```

定位到最后mMainStack.startActivityLocked，此时进入我们的Activity启动流程

记录一个大体流程，方便以后深入学习的思路整理，不同版本或多或少有些出入。原文点[这里](http://www.cloudchou.com/android/post-361.html)。




