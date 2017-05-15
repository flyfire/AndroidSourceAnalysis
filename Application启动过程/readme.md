+ http://gityuan.com/2016/10/09/app-process-create-2/ 
+ http://www.jianshu.com/p/0fb33ccdf166

Activity, Service, ContentProvider, BroadcastReceiver这四大组件,在启动的过程,当其所承载的进程不存在时需要调用startProcessLocked先创建进程.
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffm58opy7dj318g0xcdhy.jpg)

###Activity

启动Activity过程: 调用startActivity,该方法经过层层调用,最终会调用ActivityStackSupervisor.java中的startSpecificActivityLocked,当activity所属进程还没启动的情况下,则需要创建相应的进程.

[-> ActivityStackSupervisor.java]
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffm5cltwfvj31i00zik25.jpg)

###Service

启动服务过程: 调用startService,该方法经过层层调用,最终会调用ActiveServices.java中的bringUpServiceLocked,当Service进程没有启动的情况(app==null), 则需要创建相应的进程. 

[-> ActiveServices.java]
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffm5f3nc5xj31i40ya13m.jpg)

###ContentProvider

ContentProvider处理过程: 调用ContentResolver.query该方法经过层层调用, 最终会调用到AMS.java中的getContentProviderImpl,当ContentProvider所对应进程不存在,则需要创建新进程.

[-> AMS.java]

```
private final ContentProviderHolder getContentProviderImpl(...) {
    ...
    ProcessRecord proc = getProcessRecordLocked(cpi.processName, cpr.appInfo.uid, false);
    if (proc != null && proc.thread != null) {
        ...  //进程已创建的case
    } else {
        proc = startProcessLocked(cpi.processName,
                    cpr.appInfo, false, 0, "content provider",
                    new ComponentName(cpi.applicationInfo.packageName,cpi.name),
                    false, false, false);
    }
    ...
}
```

###Broadcast

广播处理过程: 调用sendBroadcast,该方法经过层层调用, 最终会调用到BroadcastQueue.java中的processNextBroadcast,当BroadcastReceiver所对应的进程尚未启动，则创建相应进程. 

[-> BroadcastQueue.java]

```
final void processNextBroadcast(boolean fromMsg) {
    ...
    ProcessRecord app = mService.getProcessRecordLocked(targetProcess,
        info.activityInfo.applicationInfo.uid, false);
    if (app != null && app.thread != null) {
        ...  //进程已创建的case
        return
    }

    if ((r.curApp=mService.startProcessLocked(targetProcess,
            info.activityInfo.applicationInfo, true,
            r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
            "broadcast", r.curComponent,
            (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                    == null) {
        ...
    }
    ...
}
```

##进程启动

在ActivityManagerService.java关于启动进程有4个同名不同参数的重载方法startProcessLocked.

```
//方法 1(a)
final ProcessRecord startProcessLocked(
    String processName, ApplicationInfo info, boolean knownToBeDead,
    int intentFlags, String hostingType, ComponentName hostingName,
    boolean allowWhileBooting, boolean isolated, boolean keepIfLarge)

//方法 1(b)
final ProcessRecord startProcessLocked(
    String processName, ApplicationInfo info, boolean knownToBeDead,
    int intentFlags, String hostingType, ComponentName hostingName,
    boolean allowWhileBooting, boolean isolated, int isolatedUid,
    boolean keepIfLarge, String abiOverride, String entryPoint,
    String[] entryPointArgs, Runnable crashHandler)

//方法 2(a)
private final void startProcessLocked(
    ProcessRecord app, String hostingType, String hostingNameStr)

//方法 2(b)
private final void startProcessLocked(
    ProcessRecord app, String hostingType, String hostingNameStr,
    String abiOverride, String entryPoint, String[] entryPointArgs)
```

###启动时机

刚解说了4大组件与进程创建的调用方法，那么接下来再来说说进程创建的触发时机有哪些？
+ 单进程App：对于这种情况，那么app首次启动某个组件时，比如通过调用startActivity来启动某个app，则先会触发创建该app进程，然后再启动该Activity。此时该app进程已创建，那么后续再该app中内部启动同一个activity或者其他组件，则都不会再创建新进程（除非该app进程被系统所杀掉）。
+ 多进程App: 对于这种情况，那么每个配置过android:process属性的组件的首次启动，则都分别需要创建进程。再次启动同一个activity，其则都不会再创建新进程（除非该app进程被系统所杀掉），但如果启动的是其他组件，则还需要再次判断其所对应的进程是否存在。

大多数情况下，app都是单进程架构，对于多进程架构的app一般是通过在AndroidManifest.xml中android:process属性来实现的。

+ 当android:process属性值以”:”开头，则代表该进程是私有的，只有该app可以使用，其他应用无法访问；
+ 当android:process属性值不以”:“开头，则代表的是全局型进程，但这种情况需要注意的是进程名必须至少包含“.”字符。

看看PackageParser.java来解析AndroidManiefst.xml过程就明白进程名的命名要求：

```
public class PackageParser {
    ...
    private static String buildCompoundName(String pkg,
       CharSequence procSeq, String type, String[] outError) {
        String proc = procSeq.toString();
        char c = proc.charAt(0);
        if (pkg != null && c == ':') {
           if (proc.length() < 2) {
               //进程名至少要有2个字符
               return null;
           }
           String subName = proc.substring(1);
           //此时并不要求强求 字符'.'作为分割符号
           String nameError = validateName(subName, false, false);
           if (nameError != null) {
               return null;
           }
           return (pkg + proc).intern();
        }
        //此时必须字符'.'作为分割符号
        String nameError = validateName(proc, true, false);
        if (nameError != null && !"system".equals(proc)) {
           return null;
        }
        return proc.intern();
    }

    private static String validateName(String name, boolean requireSeparator,
    boolean requireFilename) {
        final int N = name.length();
        boolean hasSep = false;
        boolean front = true;
        for (int i=0; i<N; i++) {
            final char c = name.charAt(i);
            if ((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z')) {
                front = false;
                continue;
            }
            if (!front) {
                if ((c >= '0' && c <= '9') || c == '_') {
                    continue;
                }
            }
            //字符'.'作为分割符号
            if (c == '.') {
                hasSep = true;
                front = true;
                continue;
            }
            return "bad character '" + c + "'";
        }
        if (requireFilename && !FileUtils.isValidExtFilename(name)) {
            return "Invalid filename";
        }
        return hasSep || !requireSeparator
                ? null : "must have at least one '.' separator";
    }
}
```

看完上面的源码.很显然对于android:process属性值不以”:“开头的进程名必须至少包含“.”字符。

###进程创建

进程的创建过程交由系统进程system_server来完成的.
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffm5r1d3q8j30mf0ec0th.jpg)

##进程启动全过程

###AMS.startProcessLocked

```
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
        boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
        boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
        String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
    long startTime = SystemClock.elapsedRealtime();
    ProcessRecord app;
    if (!isolated) {
        //根据进程名和uid检查相应的ProcessRecord
        app = getProcessRecordLocked(processName, info.uid, keepIfLarge);

        if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {
            //如果当前处于后台进程，检查当前进程是否处于bad进程列表
            if (mBadProcesses.get(info.processName, info.uid) != null) {
                return null;
            }
        } else {
            //当用户明确地启动进程，则清空crash次数，以保证其不处于bad进程直到下次再弹出crash对话框。
            mProcessCrashTimes.remove(info.processName, info.uid);
            if (mBadProcesses.get(info.processName, info.uid) != null) {
                mBadProcesses.remove(info.processName, info.uid);
                if (app != null) {
                    app.bad = false;
                }
            }
        }
    } else {
        //对于孤立进程，无法再利用已存在的进程
        app = null;
    }

    //当存在ProcessRecord,且已分配pid(正在启动或者已经启动),
    // 且caller并不认为该进程已死亡或者没有thread对象attached到该进程.则不应该清理该进程
    if (app != null && app.pid > 0) {
        if (!knownToBeDead || app.thread == null) {
            //如果这是进程中新package，则添加到列表
            app.addPackage(info.packageName, info.versionCode, mProcessStats);
            return app;
        }
        //当ProcessRecord已经被attached到先前的一个进程，则杀死并清理该进程
        killProcessGroup(app.info.uid, app.pid);
        handleAppDiedLocked(app, true, true);
    }

    String hostingNameStr = hostingName != null? hostingName.flattenToShortString() : null;
    if (app == null) {
        // 创建新的Process Record对象
        app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
        if (app == null) {
            return null;
        }
        app.crashHandler = crashHandler;
    } else {
        //如果这是进程中新package，则添加到列表
        app.addPackage(info.packageName, info.versionCode, mProcessStats);
    }
    //当系统未准备完毕，则将当前进程加入到mProcessesOnHold
    if (!mProcessesReady && !isAllowedWhileBooting(info) && !allowWhileBooting) {
        if (!mProcessesOnHold.contains(app)) {
            mProcessesOnHold.add(app);
        }
        return app;
    }
    // 启动进程【见小节3.2】
    startProcessLocked(app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
    return (app.pid != 0) ? app : null;
}
```

```
private final void startProcessLocked(ProcessRecord app, String hostingType,
        String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    long startTime = SystemClock.elapsedRealtime();
    //当app的pid大于0且不是当前进程的pid，则从mPidsSelfLocked中移除该app.pid
    if (app.pid > 0 && app.pid != MY_PID) {
        synchronized (mPidsSelfLocked) {
            mPidsSelfLocked.remove(app.pid);
            mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
        }
        app.setPid(0);
    }
    //从mProcessesOnHold移除该app
    mProcessesOnHold.remove(app);
    updateCpuStats(); //更新cpu统计信息
    try {
        try {
            if (AppGlobals.getPackageManager().isPackageFrozen(app.info.packageName)) {
                //当前package已被冻结,则抛出异常
                throw new RuntimeException("Package " + app.info.packageName + " is frozen!");
            }
        } catch (RemoteException e) {
            throw e.rethrowAsRuntimeException();
        }
        int uid = app.uid;
        int[] gids = null;
        int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
        if (!app.isolated) {
            int[] permGids = null;
            try {
                //通过Package Manager获取gids
                final IPackageManager pm = AppGlobals.getPackageManager();
                permGids = pm.getPackageGids(app.info.packageName, app.userId);
                MountServiceInternal mountServiceInternal = LocalServices.getService(
                        MountServiceInternal.class);
                mountExternal = mountServiceInternal.getExternalStorageMountMode(uid,
                        app.info.packageName);
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }

            //添加共享app和gids，用于app直接共享资源
            if (ArrayUtils.isEmpty(permGids)) {
                gids = new int[2];
            } else {
                gids = new int[permGids.length + 2];
                System.arraycopy(permGids, 0, gids, 2, permGids.length);
            }
            gids[0] = UserHandle.getSharedAppGid(UserHandle.getAppId(uid));
            gids[1] = UserHandle.getUserGid(UserHandle.getUserId(uid));
        }

        //根据不同参数,设置相应的debugFlags
        ...

        app.gids = gids;
        app.requiredAbi = requiredAbi;
        app.instructionSet = instructionSet;

        boolean isActivityProcess = (entryPoint == null);
        if (entryPoint == null) entryPoint = "android.app.ActivityThread";
        //请求Zygote创建新进程[见3.3]
        Process.ProcessStartResult startResult = Process.start(entryPoint,
                app.processName, uid, uid, gids, debugFlags, mountExternal,
                app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                app.info.dataDir, entryPointArgs);

        ...
        if (app.persistent) {
            Watchdog.getInstance().processStarted(app.processName, startResult.pid);
        }
        //重置ProcessRecord的成员变量
        app.setPid(startResult.pid);
        app.usingWrapper = startResult.usingWrapper;
        app.removed = false;
        app.killed = false;
        app.killedByAm = false;

        //将新创建的进程加入到mPidsSelfLocked
        synchronized (mPidsSelfLocked) {
            this.mPidsSelfLocked.put(startResult.pid, app);
            if (isActivityProcess) {
                Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                msg.obj = app;
                //延迟发送消息PROC_START_TIMEOUT_MSG
                mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                        ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
            }
        }
    } catch (RuntimeException e) {
        app.setPid(0); //进程创建失败,则重置pid
    }
}
```

关于Process.start()是通过socket通信告知Zygote创建fork子进程，创建新进程后将ActivityThread类加载到新进程，并调用ActivityThread.main()方法。

![](http://ww2.sinaimg.cn/large/006tNc79ly1ffm61wjo9tj30nb0dujrv.jpg)

+ App发起进程：当从桌面启动应用，则发起进程便是Launcher所在进程；当从某App内启动远程进程，则发送进程便是该App所在进程。发起进程先通过binder发送消息给system_server进程；
+ system_server进程：调用Process.start()方法，通过socket向zygote进程发送创建新进程的请求；
+ zygote进程：在执行ZygoteInit.main()后便进入runSelectLoop()循环体内，当有客户端连接时便会执行ZygoteConnection.runOnce()方法，再经过层层调用后fork出新的应用进程；
+ 新进程：执行handleChildProc方法，最后调用ActivityThread.main()方法。

###ActivityThread.main

```
public static void main(String[] args) {
    //性能统计默认是关闭的
    SamplingProfilerIntegration.start();
    //将当前进程所在userId赋值给sCurrentUser
    Environment.initForCurrentUser();

    EventLogger.setReporter(new EventLoggingReporter());
    AndroidKeyStoreProvider.install();

    //确保可信任的CA证书存放在正确的位置
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    Process.setArgV0("<pre-initialized>");

    //创建主线程的Looper对象, 该Looper是不运行退出
    Looper.prepareMainLooper();

    //创建ActivityThread对象
    ActivityThread thread = new ActivityThread();

    //建立Binder通道 【见流程3.4】
    thread.attach(false);
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    // 当设置为true时，可打开消息队列的debug log信息
    if (false) {
        Looper.myLooper().setMessageLogging(new LogPrinter(Log.DEBUG, "ActivityThread"));
    }
    Looper.loop(); //消息循环运行
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
+ 创建主线程的Looper对象: 该Looper是不运行退出. 也就是说主线程的Looper是在进程创建完成时自动创建完成,如果子线程也需要创建handler通信过程,那么就需要手动创建Looper对象,并且每个线程只能创建一次.
+ 创建ActivityThread对象thread = new ActivityThread(): 该过程会初始化几个很重要的变量:
  + mAppThread = new ApplicationThread()
  + mLooper = Looper.myLooper()
  + mH = new H(), H继承于Handler;用于处理组件的生命周期.
+ attach过程是当前主线程向system_server进程通信的过程, 将thread信息告知AMS.接下来还会进一步说明该过程.
+ sMainThreadHandler通过getHandler(),获取的对象便是mH,这就是主线程的handler对象.

之后主线程调用Looper.loop(),进入消息循环状态, 当没有消息时主线程进入休眠状态, 一旦有消息到来则唤醒主线程并执行相关操作.

###ActivityThread.attach

```
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
         //开启虚拟机的jit即时编译功能
        ViewRootImpl.addFirstDrawHandler(new Runnable() {
            @Override
            public void run() {
                ensureJitEnabled();
            }
        });
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>", UserHandle.myUserId());

        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        //创建ActivityManagerProxy对象
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        try {
            //调用基于IActivityManager接口的Binder通道【见流程3.5】
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
        }

        //观察是否快接近heap的上限
        BinderInternal.addGcWatcher(new Runnable() {
            @Override public void run() {
                if (!mSomeActivitiesChanged) {
                    return;
                }
                Runtime runtime = Runtime.getRuntime();
                long dalvikMax = runtime.maxMemory();
                long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                if (dalvikUsed > ((3*dalvikMax)/4)) {
                    mSomeActivitiesChanged = false;
                    try {
                        //当已用内存超过最大内存的3/4,则请求释放内存空间
                        mgr.releaseSomeActivities(mAppThread);
                    } catch (RemoteException e) {
                    }
                }
            }
        });
    } else {
        ...
    }
    //添加dropbox日志到libcore
    DropBox.setReporter(new DropBoxReporter());

    //添加Config回调接口
    ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
        @Override
        public void onConfigurationChanged(Configuration newConfig) {
            synchronized (mResourcesManager) {
                if (mResourcesManager.applyConfigurationToResourcesLocked(newConfig, null)) {
                    if (mPendingConfiguration == null ||
                            mPendingConfiguration.isOtherSeqNewer(newConfig)) {
                        mPendingConfiguration = newConfig;
                        sendMessage(H.CONFIGURATION_CHANGED, newConfig);
                    }
                }
            }
        }
        @Override
        public void onLowMemory() {
        }
        @Override
        public void onTrimMemory(int level) {
        }
    });
}
```

对于非系统attach的处理流程:
+ 创建线程来开启虚拟机的jit即时编译;
+ 通过binder, 调用到AMS.attachApplication, 其参数mAppThread的数据类型为ApplicationThread
+ 观察是否快接近heap的上限,当已用内存超过最大内存的3/4,则请求释放内存空间
+ 添加dropbox日志到libcore
+ 添加Config回调接口

###ActivityManagerProxy.attachApplication

```
public void attachApplication(IApplicationThread app) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(app.asBinder());
    mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0); //【见流程3.6】
    reply.readException();
    data.recycle();
    reply.recycle();
}
```
###ActivityManagerNative.onTransact

```
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    ...
     case ATTACH_APPLICATION_TRANSACTION: {
        data.enforceInterface(IActivityManager.descriptor);
        //获取ApplicationThread的binder代理类 ApplicationThreadProxy
        IApplicationThread app = ApplicationThreadNative.asInterface(
                data.readStrongBinder());
        if (app != null) {
            attachApplication(app); //此处是ActivityManagerService类中的方法 【见流程3.7】
        }
        reply.writeNoException();
        return true;
    }
    }
}
```

###ActivityManagerService.attachApplication

```
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid); // 【见流程3.8】
        Binder.restoreCallingIdentity(origId);
    }
}
```

此处的thread便是ApplicationThreadProxy对象,用于跟前面通过Process.start()所创建的进程中ApplicationThread对象进行通信.

###ActivityManagerService.attachApplicationLocked

```
private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid) {
    ProcessRecord app;
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid); // 根据pid获取ProcessRecord
        }
    } else {
        app = null;
    }
    if (app == null) {
        if (pid > 0 && pid != MY_PID) {
            //ProcessRecord为空，则杀掉该进程
            Process.killProcessQuiet(pid);
        } else {
            //退出新建进程的Looper
            thread.scheduleExit();
        }
        return false;
    }

    //还刚进入attach过程,此时thread应该为null,若不为null则表示该app附到上一个进程，则立刻清空
    if (app.thread != null) {
        handleAppDiedLocked(app, true, true);
    }

    final String processName = app.processName;
    try {
        //绑定死亡通知
        AppDeathRecipient adr = new AppDeathRecipient(app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        app.resetPackageList(mProcessStats);
        startProcessLocked(app, "link fail", processName); //重新启动进程
        return false;
    }

    //重置进程信息
    app.makeActive(thread, mProcessStats); //执行完该语句,则app.thread便不再为空
    app.curAdj = app.setAdj = -100;
    app.curSchedGroup = app.setSchedGroup = Process.THREAD_GROUP_DEFAULT;
    app.forcingToForeground = null;
    updateProcessForegroundLocked(app, false, false);
    app.hasShownUi = false;
    app.debugging = false;
    app.cached = false;
    app.killedByAm = false;
    mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app); //移除进程启动超时的消息

    //系统处于ready状态或者该app为FLAG_PERSISTENT进程,则为true
    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

    //app进程存在正在启动中的provider,则超时10s后发送CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG消息
    if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
        Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
        msg.obj = app;
        mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
    }

    try {
        ...
        ensurePackageDexOpt(app.instrumentationInfo != null
                ? app.instrumentationInfo.packageName
                : app.info.packageName);

        //获取应用appInfo
        ApplicationInfo appInfo = app.instrumentationInfo != null
                ? app.instrumentationInfo : app.info;
        ...
        // 绑定应用 [见流程3.9]
        thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                isRestrictedBackupMode || !normalMode, app.persistent,
                new Configuration(mConfiguration), app.compat,
                getCommonServicesLocked(app.isolated),
                mCoreSettingsObserver.getCoreSettingsLocked());
        //更新进程LRU队列
        updateLruProcessLocked(app, false, null);
        app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
    } catch (Exception e) {
        app.resetPackageList(mProcessStats);
        app.unlinkDeathRecipient();
        //每当bind操作失败,则重启启动进程, 此处有可能会导致进程无限重启
        startProcessLocked(app, "bind fail", processName);
        return false;
    }

    mPersistentStartingProcesses.remove(app);
    mProcessesOnHold.remove(app);
    boolean badApp = false;
    boolean didSomething = false;

    //Activity: 检查最顶层可见的Activity是否等待在该进程中运行
    if (normalMode) {
        try {
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            badApp = true;
        }
    }

    //Service: 寻找所有需要在该进程中运行的服务
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
        } catch (Exception e) {
            badApp = true;
        }
    }

    //Broadcast: 检查是否在这个进程中有下一个广播接收者
    if (!badApp && isPendingBroadcastProcessLocked(pid)) {
        try {
            didSomething |= sendPendingBroadcastsLocked(app);
        } catch (Exception e) {
            badApp = true;
        }
    }
    //检查是否在这个进程中有下一个backup代理
    if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
        ensurePackageDexOpt(mBackupTarget.appInfo.packageName);
        try {
            thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                    compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                    mBackupTarget.backupMode);
        } catch (Exception e) {
            badApp = true;
        }
    }
    if (badApp) { //杀掉bad应用
        app.kill("error during init", true);
        handleAppDiedLocked(app, false, true);
        return false;
    }
    if (!didSomething) {
        updateOomAdjLocked(); //更新adj的值
    }
    return true;
}
```
+ 根据pid从mPidsSelfLocked中查询到相应的ProcessRecord对象app;
+ 当app==null,意味着本次创建的进程不存在, 则直接返回.
+ 还刚进入attach过程,此时thread应该为null,若不为null则表示该app附到上一个进程，则调用handleAppDiedLocked清理.
+ 绑定死亡通知,当进程pid死亡时会通过binder死亡回调,来通知system_server进程死亡的消息;
+ 重置ProcessRecord进程信息, 此时app.thread也赋予了新值,便不再为空.
+ app进程存在正在启动中的provider,则超时10s后发送CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG消息
+ 调用thread.bindApplication绑定应用进程, 后面再进一步说明
+ 处理Provider, Activity, Service, Broadcast相应流程

###ApplicationThreadProxy.bindApplication

[-> ApplicationThreadNative.java ::ApplicationThreadProxy]

```
class ApplicationThreadProxy implements IApplicationThread {
    ...
    public final void bindApplication(String packageName, ApplicationInfo info,
            List<ProviderInfo> providers, ComponentName testName, ProfilerInfo profilerInfo,
            Bundle testArgs, IInstrumentationWatcher testWatcher,
            IUiAutomationConnection uiAutomationConnection, int debugMode,
            boolean openGlTrace, boolean restrictedBackupMode, boolean persistent,
            Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
            Bundle coreSettings) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeString(packageName);
        info.writeToParcel(data, 0);
        data.writeTypedList(providers);
        if (testName == null) {
            data.writeInt(0);
        } else {
            data.writeInt(1);
            testName.writeToParcel(data, 0);
        }
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        data.writeBundle(testArgs);
        data.writeStrongInterface(testWatcher);
        data.writeStrongInterface(uiAutomationConnection);
        data.writeInt(debugMode);
        data.writeInt(openGlTrace ? 1 : 0);
        data.writeInt(restrictedBackupMode ? 1 : 0);
        data.writeInt(persistent ? 1 : 0);
        config.writeToParcel(data, 0);
        compatInfo.writeToParcel(data, 0);
        data.writeMap(services);
        data.writeBundle(coreSettings);
        mRemote.transact(BIND_APPLICATION_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    ...
}
```

ApplicationThreadProxy经过binder ipc传递到ApplicationThreadNative的onTransact过程.

###ApplicationThreadNative.onTransact

```
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    ...
    case BIND_APPLICATION_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        String packageName = data.readString();
        ApplicationInfo info =
            ApplicationInfo.CREATOR.createFromParcel(data);
        List<ProviderInfo> providers =
            data.createTypedArrayList(ProviderInfo.CREATOR);
        ComponentName testName = (data.readInt() != 0)
            ? new ComponentName(data) : null;
        ProfilerInfo profilerInfo = data.readInt() != 0
                ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
        Bundle testArgs = data.readBundle();
        IBinder binder = data.readStrongBinder();
        IInstrumentationWatcher testWatcher = IInstrumentationWatcher.Stub.asInterface(binder);
        binder = data.readStrongBinder();
        IUiAutomationConnection uiAutomationConnection =
                IUiAutomationConnection.Stub.asInterface(binder);
        int testMode = data.readInt();
        boolean openGlTrace = data.readInt() != 0;
        boolean restrictedBackupMode = (data.readInt() != 0);
        boolean persistent = (data.readInt() != 0);
        Configuration config = Configuration.CREATOR.createFromParcel(data);
        CompatibilityInfo compatInfo = CompatibilityInfo.CREATOR.createFromParcel(data);
        HashMap<String, IBinder> services = data.readHashMap(null);
        Bundle coreSettings = data.readBundle();
        //[见流程3.11]
        bindApplication(packageName, info, providers, testName, profilerInfo, testArgs,
                testWatcher, uiAutomationConnection, testMode, openGlTrace,
                restrictedBackupMode, persistent, config, compatInfo, services, coreSettings);
        return true;
    }
    ...
}
```

###ApplicationThread.bindApplication

[-> ActivityThread.java ::ApplicationThread]

```
public final void bindApplication(String processName, ApplicationInfo appInfo,
        List<ProviderInfo> providers, ComponentName instrumentationName,
        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
        IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
        Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
        Bundle coreSettings) {

    if (services != null) {
        //将services缓存起来, 减少binder检索服务的次数
        ServiceManager.initServiceCache(services);
    }

    //发送消息H.SET_CORE_SETTINGS [见小节3.12]
    setCoreSettings(coreSettings);

    IPackageManager pm = getPackageManager();
    android.content.pm.PackageInfo pi = null;
    try {
        pi = pm.getPackageInfo(appInfo.packageName, 0, UserHandle.myUserId());
    } catch (RemoteException e) {
    }
    if (pi != null) {
        boolean sharedUserIdSet = (pi.sharedUserId != null);
        boolean processNameNotDefault = (pi.applicationInfo != null &&
         !appInfo.packageName.equals(pi.applicationInfo.processName));
        boolean sharable = (sharedUserIdSet || processNameNotDefault);

        if (!sharable) {
            VMRuntime.registerAppInfo(appInfo.packageName, appInfo.dataDir,
                                    appInfo.processName);
        }
    }

    //初始化AppBindData
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
    //发送消息H.BIND_APPLICATION [见小节3.13]
    sendMessage(H.BIND_APPLICATION, data);
}
```

其中setCoreSettings()过程就是调用sendMessage(H.SET_CORE_SETTINGS, coreSettings) 来向主线程发送SET_CORE_SETTINGS消息.bindApplication方法的主要功能是依次向主线程发送消息H.SET_CORE_SETTINGS和H.BIND_APPLICATION. 接下来再来说说这两个消息的处理过程

###ActivityThread.handleSetCoreSettings

[-> ActivityThread.java ::H]
当主线程收到H.SET_CORE_SETTINGS,则调用handleSetCoreSettings

```
private void handleSetCoreSettings(Bundle coreSettings) {
    synchronized (mResourcesManager) {
        mCoreSettings = coreSettings;
    }
    onCoreSettingsChange();
}

private void onCoreSettingsChange() {
    boolean debugViewAttributes = mCoreSettings.getInt(Settings.Global.DEBUG_VIEW_ATTRIBUTES, 0) != 0;
    if (debugViewAttributes != View.mDebugViewAttributes) {
        View.mDebugViewAttributes = debugViewAttributes;

        // 由于发生改变, 请求所有的activities重启启动
        for (Map.Entry<IBinder, ActivityClientRecord> entry : mActivities.entrySet()) {
            requestRelaunchActivity(entry.getKey(), null, null, 0, false, null, null, false);
        }
    }
}
```

###ActivityThread.handleBindApplication

[-> ActivityThread.java ::H]
当主线程收到H.BIND_APPLICATION,则调用handleBindApplication
```
private void handleBindApplication(AppBindData data) {

    mBoundApplication = data;
    mConfiguration = new Configuration(data.config);
    mCompatConfiguration = new Configuration(data.config);
    ...

    //设置进程名, 也就是说进程名是在进程真正创建以后的BIND_APPLICATION过程中才取名
    Process.setArgV0(data.processName);
    android.ddm.DdmHandleAppName.setAppName(data.processName, UserHandle.myUserId());

    if (data.persistent) {
        //低内存设备, persistent进程不采用硬件加速绘制,以节省内存使用量
        if (!ActivityManager.isHighEndGfx()) {
            HardwareRenderer.disable(false);
        }
    }

    //重置时区
    TimeZone.setDefault(null);
    Locale.setDefault(data.config.locale);

    //更新系统配置
    mResourcesManager.applyConfigurationToResourcesLocked(data.config, data.compatInfo);
    mCurDefaultDisplayDpi = data.config.densityDpi;
    applyCompatConfiguration(mCurDefaultDisplayDpi);

    //获取LoadedApk对象[见小节3.13.1]
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    ...

    // 创建ContextImpl上下文
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    if (!Process.isIsolated()) {
        final File cacheDir = appContext.getCacheDir();
        if (cacheDir != null) {
            System.setProperty("java.io.tmpdir", cacheDir.getAbsolutePath());
        }

        //用于存储产生/编译的图形代码
        final File codeCacheDir = appContext.getCodeCacheDir();
        if (codeCacheDir != null) {
            setupGraphicsSupport(data.info, codeCacheDir);
        }
    }
    ...

    //当处于调试模式,则运行应用生成systrace信息
    boolean appTracingAllowed = (data.appInfo.flags&ApplicationInfo.FLAG_DEBUGGABLE) != 0;
    Trace.setAppTracingAllowed(appTracingAllowed);

    //初始化 默认的http代理
    IBinder b = ServiceManager.getService(Context.CONNECTIVITY_SERVICE);
    if (b != null) {
        IConnectivityManager service = IConnectivityManager.Stub.asInterface(b);
        final ProxyInfo proxyInfo = service.getProxyForNetwork(null);
        Proxy.setHttpProxySystemProperty(proxyInfo);
    }

    if (data.instrumentationName != null) {
        ...
    } else {
        mInstrumentation = new Instrumentation();
    }

    //FLAG_LARGE_HEAP则清除内存增长上限
    if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
        dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
    } else {
        dalvik.system.VMRuntime.getRuntime().clampGrowthLimit();
    }

    try {
        // 此处data.info是指LoadedApk, 通过反射创建目标应用Application对象[见小节3.14]
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;

        if (!data.restrictedBackupMode) {
            List<ProviderInfo> providers = data.providers;
            if (providers != null) {
                installContentProviders(app, providers);
                mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
            }
        }

        mInstrumentation.onCreate(data.instrumentationArgs);
        //调用Application.onCreate()回调方法.
        mInstrumentation.callApplicationOnCreate(app);

    } finally {
        StrictMode.setThreadPolicy(savedPolicy);
    }
}
```

在handleBindApplication()的过程中,会同时设置以下两个值:
+ LoadedApk.mApplication
+ ActivityThread.mInitialApplication

###getPackageInfoNoCheck

[-> ActivityThread.java]

```
public final LoadedApk getPackageInfoNoCheck(ApplicationInfo ai,
        CompatibilityInfo compatInfo) {
    return getPackageInfo(ai, compatInfo, null, false, true, false);
}

private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,
    ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
        boolean registerPackage) {
    final boolean differentUser = (UserHandle.myUserId() != UserHandle.getUserId(aInfo.uid));
    synchronized (mResourcesManager) {
        WeakReference<LoadedApk> ref;
        if (differentUser) {
            //不支持跨用户
            ref = null;
        } else if (includeCode) {
            ref = mPackages.get(aInfo.packageName);
        } else {
            ref = mResourcePackages.get(aInfo.packageName);
        }

        LoadedApk packageInfo = ref != null ? ref.get() : null;
        if (packageInfo == null || (packageInfo.mResources != null
                && !packageInfo.mResources.getAssets().isUpToDate())) {
            //创建LoadedApk对象
            packageInfo =
                new LoadedApk(this, aInfo, compatInfo, baseLoader,
                        securityViolation, includeCode &&
                        (aInfo.flags&ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);

            if (mSystemThread && "android".equals(aInfo.packageName)) {
                packageInfo.installSystemApplicationInfo(aInfo,
                        getSystemContext().mPackageInfo.getClassLoader());
            }

            if (differentUser) {
                //不支持跨用户
            } else if (includeCode) {
                mPackages.put(aInfo.packageName,
                        new WeakReference<LoadedApk>(packageInfo));
            } else {
                mResourcePackages.put(aInfo.packageName,
                        new WeakReference<LoadedApk>(packageInfo));
            }
        }
        return packageInfo;
    }
}
```
创建LoadedApk对象
###makeApplication

[-> LoadedApk.java]
```
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
        //[见小节3.14.1]
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        //[见小节3.14.2]
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        ...
    }
    mActivityThread.mAllApplications.add(app);
    //设置mApplication对象值
    mApplication = app;

    if (instrumentation != null) {
        try {
            //调用app.onCreate()
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            ...
        }
    }

    SparseArray<String> packageIdentifiers = getAssets(mActivityThread)
            .getAssignedPackageIdentifiers();
    final int N = packageIdentifiers.size();
    for (int i = 0; i < N; i++) {
        final int id = packageIdentifiers.keyAt(i);
        if (id == 0x01 || id == 0x7f) {
            continue;
        }
        //重写所有apk库中的R常量[见小节3.14.3]
        rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
    }
    return app;
}
```

### createAppContext

[-> ContextImpl.java]
```
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    return new ContextImpl(null, mainThread,
            packageInfo, null, null, false, null, null, Display.INVALID_DISPLAY);
}
```

###newApplication

[-> Instrumentation.java]
```
public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException,
        ClassNotFoundException {
    return newApplication(cl.loadClass(className), context);
}
```
创建Application对象, 该对象名来自于mApplicationInfo.className.

###rewriteRValues

[-> LoadedApk.java] 

```
try { rClazz = cl.loadClass(packageName + “.R”); } 
catch (ClassNotFoundException e) { return; }
final Method callback;
    try {
        callback = rClazz.getMethod("onResourcesLoaded", int.class);
    } catch (NoSuchMethodException e) {
        return;
    }

    Throwable cause;
    try {
        callback.invoke(null, id);
        return;
    } catch (IllegalAccessException e) {
        cause = e;
    } catch (InvocationTargetException e) {
        cause = e.getCause();
    }

    throw new RuntimeException("Failed to rewrite resource references for " + packageName,
            cause);
}
```

###总结

![](http://ww1.sinaimg.cn/large/006tNc79ly1ffm6org293j30l90jwjsf.jpg)


```
ActivityStackSupervisor->ActivityManagerService:startProcessLocked(.)
ActivityManagerService->ActivityManagerService:startProcessLocked(..)
ActivityManagerService->Process:start()
Process->Process:startViaZygote()
Process->Process:zygoteSendArgsAndGetResult()
Process->ZygoteInit:socket
ZygoteInit->ZygoteInit:runSelectLoop()
ZygoteInit->ZygoteConnection:runOnce()
ZygoteConnection->ZygoteConnection:handleChildProc()
ZygoteConnection->RuntimeInit:zygoteInit()
RuntimeInit->RuntimeInit:applicationInit()
RuntimeInit->RuntimeInit:invokeStaticMain()
RuntimeInit->ActivityThread:main()
ActivityThread->ActivityThread:attach()
ActivityThread->ActivityManagerService:attachApplication()
ActivityManagerService->ActivityManagerService:attachApplicationLocked()
ActivityManagerService->ActivityStackSupervisor:attachApplicationLocked()
ActivityStackSupervisor->ActivityManagerService:realStartActivityLocked()
```




