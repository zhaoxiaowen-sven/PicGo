 一、 startService 和 bindService的区别
    ![](https://leanote.com/api/file/getImage?fileId=59341fcbab64415b02001748)

 - 执行startService时，Service会经历onCreate->onStartCommand。当执行stopService时，直接调用onDestroy方法。调用者如果没有stopService，Service会一直在后台运行，下次调用者再起来仍然可以stopService。
 - 执行bindService时，Service会经历onCreate->onBind。这个时候调用者和Service绑定在一起。调用者调用unbindService方法或者调用者Context不存在了（如Activity被finish了），Service就会调用onUnbind->onDestroy。这里所谓的绑定在一起就是说两者共存亡了。
 - 多次调用startService，该Service只能被创建一次，即该Service的onCreate方法只会被调用一次。但是每次调用startService，onStartCommand方法都会被调用。Service的onStart方法在API5时被废弃，替代它的是onStartCommand方法。
 - 第一次执行bindService时，onCreate和onBind方法会被调用，但是多次执行bindService时，onCreate和onBind方法并不会被多次调用，即并不会多次创建服务和绑定服务。


二、startService的过程
[http://www.jianshu.com/p/c0aadd5bf7a5](http://1)
[http://wujingchao.com/2016/02/10/art-of-android-development-notes-startservice/](http://2)

1.frameworks/base/core/java/android/content/ContextWrapper.java

    public ComponentName startService(Intent service) {
            warnIfCallingFromSystemProcess();
            return startServiceCommon(service, mUser);
        }
    private ComponentName startServiceCommon(Intent service, UserHandle user) {
        ComponentName cn = ActivityManagerNative.getDefault().startService(
            mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                        getContentResolver()), getOpPackageName(), user.getIdentifier());    
    ...
    }
2.通过ActivityManagerNative.getDefault().startService的ActivityManagerNative->ActivityManagerService->ActiveServices.startServiceLocked

    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
    int callingPid, int callingUid, String callingPackage, int userId)
    throws TransactionTooLargeException {
    …
    //PackageManagerService解析出Intent得到要启动的ServiceRecord
    ServiceLookupResult res =
    retrieveServiceLocked(service, resolvedType, callingPackage,
    callingPid, callingUid, userId, true, callerFg);
        ServiceRecord r = res.record;            
    ... 
    return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    }
    ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
    boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
    …
    String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false);
    …
    return r.name;
    }
    private final String bringUpServiceLocked(ServiceRecord r,
    int intentFlags, boolean execInFg, boolean whileRestarting) {
    //如果进程已经存在的情况下就不是处理下面的流程，直接处理onStart的流程
    if (r.app != null && r.app.thread != null) {
    sendServiceArgsLocked(r, execInFg, false);
    return null;
    }
    //处于restart的状态(在onStartCommand里面处理了服务被杀之后的行为)也不会处理
    if (!whileRestarting && r.restartDelay > 0) {
    return null;
    }
    if (mRestartingServices.remove(r)) {
    clearRestartingIfNeededLocked(r);
    }
    if (r.delayed) {
    getServiceMap(r.userId).mDelayedStartList.remove(r);
    r.delayed = false;
    }
    //…
    final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
    final String procName = r.processName;
    ProcessRecord app;
    //独立的进程运行isolated为true,
    if (!isolated) {
    app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
    if (app != null && app.thread != null) {
    try {
    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
    //直接启动服务，不用开启新的进程
    realStartServiceLocked(r, app, execInFg);
    return null;
    } catch (RemoteException e) {
    Slog.w(TAG, “Exception when starting service ” + r.shortName, e);
    }
    }
    } else {
    app = r.isolatedProc;
    }
    第一次创建时，走到这里，先创建一个service的进程
    if (app == null) {
        //开启新的进程 
        if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                "service", r.name, false, isolated, false))== null) {
            bringDownServiceLocked(r);
            return msg;
        }
        if (isolated) {
            r.isolatedProc = app;
        }
    }
    //将ServiceRecord加入即将启动mPendingServices列表里，后面进程启动成功后在启动Service
    if (!mPendingServices.contains(r)) {
        mPendingServices.add(r);
    }
    //...
    return null;
    }
如果进程已经存在的情况下就不是处理下面的流程，直接调用realStartServiceLocked处理onStart的流程。ActivityManagerService.startProcessLocked开启进程，procName为AndroidManifest中Service标签了process指定的进程名，默认是包名。

    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
    boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
    boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
    String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
        ProcessRecord app;
        if (!isolated) {
        app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
        } else {
            // If this is an isolated process, it can't re-use an existing process.
            app = null;
        }
        //...
        String hostingNameStr = hostingName != null
                ? hostingName.flattenToShortString() : null;
       //...
       if (app == null) {
            //构建一个新的的ProcessRecord
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
            app.crashHandler = crashHandler;
            mProcessNames.put(processName, app.uid, app);
            if (isolated) {
                mIsolatedProcesses.put(app.uid, app);
            }
        } 
        //...
        startProcessLocked(
                app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
        return (app.pid != 0) ? app : null;
    }


构建新的ProcessRecord，startProcessLocked开启进程:

    private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        //...
        try {
            int uid = app.uid;
            int[] gids = null;
            int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
            if (!app.isolated) {
                //...
            }
            int debugFlags = 0;
            //...
            boolean isActivityProcess = (entryPoint == null);
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            //...
    }
调用Process.start开启一个新的进程，新进程的入口点就是android.app.ActivityThread，执行里面的main方法。

     public final class ActivityThread {
        final ApplicationThread mAppThread = new ApplicationThread();
        public static void main(String[] args) {
            //...
            Looper.prepareMainLooper();
            ActivityThread thread = new ActivityThread();
            thread.attach(false);
            if (sMainThreadHandler == null) {
                sMainThreadHandler = thread.getHandler();
            }
            Looper.loop();
            throw new RuntimeException("Main thread loop unexpectedly exited");
        }
         private void attach(boolean system) {
            //...
            if (!system) {
                //...
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                try {
                    mgr.attachApplication(mAppThread);
                } catch (RemoteException ex) {
                    // Ignore
                }
                //...
            } 
        }
    }
准备主线程的Looper，调用attachApplication通知ActiviyManangerService主线程准备完毕，然后loop开始消息循环。
ApplicationThread通过IPC向ActivityManagerService调用attachApplication，并传递mAppThread给ActivityManagerService，mAppThread是一个Binder对象，用于ActivityManagerService向我们发起调用。注意从这里开始已经是在新进程里面执行了。
ApplicationThread对象继承自ApplicationThreadNative.java，在ActivityThread对象被创建时，它也被构造了，我前面已经提到过了，它继承了ApplicationThreadNative类，熟悉进程通信代理机制的朋友就清楚了，ApplicationThread就是一个通信代理存根实现类，我们可以看它的实现方法，都是调用queueOrSendMessage方法，派发消息交给ActivityThread的mH去处理，那么我们很清楚了，ActivityThread代理存根对象，它负责执行来自远程的调用，这些远程的调用大部分来自system_process，所以，system_process很容易通过ApplicationThread的客户端代理对象控制ActivityThread，事实就是如此，后面我们可以很好地看到这一点

        base/core/java/android/app/ActivityManagerNative.java
        
        class ActivityManagerProxy implements IActivityManager{
        public void attachApplication(IApplicationThread app) throws RemoteException
        {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            data.writeInterfaceToken(IActivityManager.descriptor);
            data.writeStrongBinder(app.asBinder());
            mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
            reply.readException();
            data.recycle();
            reply.recycle();
        }
    }
遂进入到ActivityManagerService:

    public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid);
        Binder.restoreCallingIdentity(origId);
    }
attachApplicationLocked

    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        ProcessRecord app;
        ...
        final String processName = app.processName;
        ...
        app.makeActive(thread, mProcessStats);
        app.curAdj = app.setAdj = -100;
        app.curSchedGroup = app.setSchedGroup = Process.THREAD_GROUP_DEFAULT;
        app.forcingToForeground = null;
        updateProcessForegroundLocked(app, false, false);
        app.hasShownUi = false;
        app.debugging = false;
        app.cached = false;
        app.killedByAm = false;
        ...
         //使用ApplicationThread发起IPC调用bindApplication ->ActivityThread．bindApplication:
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
        ...   
            
        boolean badApp = false;
        //...
        // Find any services that should be running in this process...
        if (!badApp) {
            try {
                //接着调用ActivieServices的attachApplicationLocked通知客户端启动Service:
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
                badApp = true;
            }
        }
       //...
        return true;
    }

使用ApplicationThread发起IPC调用bindApplication ->ActivityThread.bindApplication:

    public final void bindApplication(String processName, ApplicationInfo appInfo,
                    List<ProviderInfo> providers, ComponentName instrumentationName,
                    ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                    IInstrumentationWatcher instrumentationWatcher,
                    IUiAutomationConnection instrumentationUiConnection, int debugMode,
                    boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
                    Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
                    Bundle coreSettings) {
                //... init
                IPackageManager pm = getPackageManager();
                android.content.pm.PackageInfo pi = null;
                try {
                    pi = pm.getPackageInfo(appInfo.packageName, 0, UserHandle.myUserId());
                } catch (RemoteException e) {
                }
                if (pi != null) {
                   //处理sharedUid的情况
                   //...
                }
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
    private void sendMessage(int what, Object obj) {
            sendMessage(what, obj, 0, 0, false);
    }
    final H mH = new H();
    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }


 主线程的Looper前面已经在ActivityThread主线程里面初始化了，然后向Handler发消息实现进程切换(因为bindApplication是在客户端Binder线程池里面调用的)。

     private class H extends Handler {
             public void handleMessage(Message msg) {
                //...
                 case BIND_APPLICATION:
                            AppBindData data = (AppBindData)msg.obj;
                            handleBindApplication(data);
                            break;
             }
          }

接着调用ActivityThread的handleBindApplication，主要是然客户端初始化应用程序的一些状态比如时区地域，Instrumentation，LoadedApk等等。

    private void handleBindApplication(AppBindData data) {
        mBoundApplication = data;
        mConfiguration = new Configuration(data.config);
        mCompatConfiguration = new Configuration(data.config);
        //...
        TimeZone.setDefault(null);
        //...
        Locale.setDefault(data.config.locale);
        //...
    }
再回到ActivityManagerService中的attachApplicationLocked，接着调用ActivieServices的attachApplicationLocked通知客户端启动Service

        if (!badApp) {
            try {
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
                badApp = true;
            }
        }
attachApplicationLocked，mPendingServices就是前面加入列表的ServiceRecord，过滤要启动的ServiceRecord，调用realStartServiceLocked:   

    boolean attachApplicationLocked(ProcessRecord proc, String processName)
            throws RemoteException {
        boolean didSomething = false;
        // Collect any services that are waiting for this process to come up.
        if (mPendingServices.size() > 0) {
            ServiceRecord sr = null;
            try {
                for (int i=0; i<mPendingServices.size(); i++) { sr = mPendingServices.get(i); //过滤我们客户端当前的进程 if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid || !processName.equals(sr.processName))) { continue; } mPendingServices.remove(i); i--; proc.addPackage(sr.appInfo.packageName, sr.appInfo.versionCode, mAm.mProcessStats); realStartServiceLocked(sr, proc, sr.createdFromFg); didSomething = true; } } catch (RemoteException e) { Slog.w(TAG, "Exception in new application when starting service " - sr.shortName, e); throw e; } } if (mRestartingServices.size()> 0) {
           //处理restart的状态
           //...
        }
        return didSomething;
    }
pendingStarts在放入要执行start操作的列表里面，在执行sendServiceArgsLocked告诉客户端执行onStart:

    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        //..
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();
        app.services.add(r);
        //将Service加入到正在执行的executingServices(ProcessRecord)列表里
        bumpServiceExecutingLocked(r, execInFg, "create");
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked();
        boolean created = false;
        try {
            //...
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            //前台进程显示Notification
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
          //...
      }
        // If the service is in the started state, and there are no
        // pending arguments, then fake up one so its onStartCommand() will
        // be called.
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }
        sendServiceArgsLocked(r, execInFg, true);
        //...
    }
 先看ActivityThread的scheduleCreateService,这里对应的token就是ActivityManagerService创建的ServiceRecord，ServiceInfo是ActivityManagerService为我们解析AndroidManifest的Service标签:

    public final void scheduleCreateService(IBinder token,
            ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
        updateProcessState(processState, false);
        CreateServiceData s = new CreateServiceData();
        s.token = token;
        s.info = info;
        s.compatInfo = compatInfo;
        sendMessage(H.CREATE_SERVICE, s);
    }
同样是向Handler发送消息实现进程切换:

     private class H extends Handler {
        public void handleMessage(Message msg) {
            //...
             case CREATE_SERVICE:
                handleCreateService((CreateServiceData)msg.obj);
                break;
            //...
        }
    }
执行ActivityThread的handleCreateService，实现创建服务并执行onCreate，调用ActivityManagerService的serviceDoneExecuting，onCreate更新下Service的一些状态

     private void handleCreateService(CreateServiceData data) {
        ...
            Service service = null;
            ...
                java.lang.ClassLoader cl = packageInfo.getClassLoader();
                service = (Service) cl.loadClass(data.info.name).newInstance();
                ...
                service.attach(context, this, data.info.name, data.token, app,
                        ActivityManagerNative.getDefault());
                service.onCreate();
                mServices.put(data.token, service);
                ...
                //serviceDoneExecuting的主要工作是当service启动完成，则移除service Timeout消息。
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                ...
        }

再回到上面的ActivieServices的sendServiceArgsLocked告诉客户端要执行onStartCommand，将要执行的onStart的参数（例如startId）传回客户端:

        private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) {
        //...
        while (r.pendingStarts.size() > 0) {
            try {
                ServiceRecord.StartItem si = r.pendingStarts.remove(0);
                //...
                si.deliveredTime = SystemClock.uptimeMillis();
                r.deliveredStarts.add(si);
                si.deliveryCount++;
                //更新正在执行的状态
                bumpServiceExecutingLocked(r, execInFg, "start");
                int flags = 0;
                if (si.deliveryCount > 1) {
                    flags |= Service.START_FLAG_RETRY;
                }
                if (si.doneExecutingCount > 0) {
                    flags |= Service.START_FLAG_REDELIVERY;
                }
                //IPC过程 ApplicationThreadNative -> ActivityThread
                r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
            } catch (RemoteException e) {
                //...
            } 
        }
    }
接着执行ActivityThread里的scheduleServiceArgs:

    public final void scheduleServiceArgs(IBinder token, boolean taskRemoved, int startId,
                int flags ,Intent args) {
                ServiceArgsData s = new ServiceArgsData();
                s.token = token;
                s.taskRemoved = taskRemoved;
                s.startId = startId;
                s.flags = flags;
                s.args = args;
                sendMessage(H.SERVICE_ARGS, s);
            }
同样发送消息给主线程执行handleServiceArgs，mServices为客户端维护的Service列表:

        private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);
        ...
                int res;
                if (!data.taskRemoved) {
                    res = s.onStartCommand(data.args, data.flags, data.startId);
               ...
               //通知AMSservice的状态
               ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
               
    }
onStartCommand执行完后会返回一个参数，用于控制Service的一些行为，例如进程被杀死之后Service的行为。
随后调用serviceDoneExecuting告诉ActivityManagerService，onStart已经执行完了，ActivityManagerService再更新一些状态，就这样Service就运行起来了。

三、bindService的过程分析
    这个过程中Service所在的进程是已经别启动了的
    [http://www.jianshu.com/p/37e0e66979a6](http://1)
    [http://blog.csdn.net/jelly_fang/article/details/50488915](http://2)
    [http://www.cnblogs.com/android-blogs/p/5718302.html](http://3)

frameworks\base\core\java\android\app\ContextImpl.java

     public boolean bindService(Intent service, ServiceConnection conn,
                int flags) {
            warnIfCallingFromSystemProcess();
            return bindServiceCommon(service, conn, flags, Process.myUserHandle());
        }
        private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
                UserHandle user) {
            IServiceConnection sd;
            ...
            1.onbind最终回调ServiceConnection的onServiceConnected方法
            if (mPackageInfo != null) {
                sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),
                        mMainThread.getHandler(), flags);
            } 
            ...
            2.AMS中bind过程
            int res = ActivityManagerNative.getDefault().bindService(
                    mMainThread.getApplicationThread(), getActivityToken(), service,
                    service.resolveTypeIfNeeded(getContentResolver()),
                    sd, flags, getOpPackageName(), user.getIdentifier());    
            ...
        }

分析1过程：
sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),
                mMainThread.getHandler(), flags);
                    
    frameworks\base\core\java\android\app\LoadedApk.java
    
    public final IServiceConnection getServiceDispatcher(ServiceConnection c,
            Context context, Handler handler, int flags) {
        synchronized (mServices) {
            //创建ServiceDispatcher
            LoadedApk.ServiceDispatcher sd = null;
            ...
            if (sd == null) {
                sd = new ServiceDispatcher(c, context, handler, flags);
                ...
            }
            ...
            //返回的是ServiceDispatcher的内部类InnerConnection
            return sd.getIServiceConnection();
        }
    }

 


   ServiceDispatcher类中的方法

    static final class ServiceDispatcher {
        private final ServiceDispatcher.InnerConnection mIServiceConnection;
        private final ServiceConnection mConnection;
        private final Context mContext;
        private final Handler mActivityThread;
        private final ServiceConnectionLeaked mLocation;
        private final int mFlags;
        ...
        //ServiceDispatcher的内部类
        private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;
    
            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }
            //保存当前激活的Serviceconnection的Map
            private final ArrayMap<ComponentName, ServiceDispatcher.ConnectionInfo> mActiveConnections
            = new ArrayMap<ComponentName, ServiceDispatcher.ConnectionInfo>();
            
            public void connected(ComponentName name, IBinder service) throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    //调用到了外部类ServiceDispatcher中的connected
                    sd.connected(name, service);
                }
            }
        }
        //ServiceDispatcher的connected方法
        public void connected(ComponentName name, IBinder service) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0));
            } else {
            //调用到ServiceDispatcher的doConnected
                doConnected(name, service);
            }
        }
        
        public void doConnected(ComponentName name, IBinder service) {
            ...
            if (service != null) {
                    // A new service is being connected... set it all up.
                    mDied = false;
                    info = new ConnectionInfo();
                    info.binder = service;
                    info.deathMonitor = new DeathMonitor(name, service);
                    ...
            }   
            ...
            // If there is a new service, it is now connected.
            if (service != null) {
                //调用了 mConnection.onServiceConnected(name, service)这个方法,后面会通过InnerConnection实例来远程回调这个方法。
                //这个mConnection其实就是一开始ServiceDispatcher 的构造函数中传进来的ServiceConnection实例。
                mConnection.onServiceConnected(name, service);
            }
        }

   


 分析2过程：
    int res = ActivityManagerNative.getDefault().bindService(
            mMainThread.getApplicationThread(), getActivityToken(), service,
            service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, getOpPackageName(), user.getIdentifier());
                
   ActivityManagerNative.getDefault() 最终都会调用ActiveServices.bindServiceLocked方法,细致的不去跟踪
   frameworks\base\services\core\java\com\android\server\am\ActiveServices.java
   s（ServiceRecord ）、b（AppBindRecord ）、c（ConnectionRecord ）、callerApp （ProcessRecord） 这四个类型的一些保存之类的操作，方便以后调用
    
    
        int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags,
            String callingPackage, int userId) throws TransactionTooLargeException {
        ...
      
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        ...
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    Binder.getCallingPid(), Binder.getCallingUid(), userId, true, callerFg);
        if (res == null) {
            return 0;
        }
        if (res.record == null) {
            return -1;
        }
        ServiceRecord s = res.record;
        ...
        try {
            mAm.startAssociationLocked(callerApp.uid, callerApp.processName,
                    s.appInfo.uid, s.name, s.processName);
            //获取一个ServiceRecord对象，如果不存在就创建        
            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
            ConnectionRecord c = new ConnectionRecord(b, activity,connection, flags, clientLabel, clientIntent);
            
            //这个binder其实可以简单的看做成IServiceConnection ，后面可以通过它asInterface方法获取binder驱动来远程调用IServiceConnection （其实就是IServiceConnection–>InnerConnection –>ServiceDispatcher ）里面的方法。然后把这个binder放到了s（ServiceRecord ）的connections的集合中,clist就是一个connectionRecord的集合，应为可能不止一个activity或者组件需要绑定这个service。
            IBinder binder = connection.asBinder();
            
            ArrayList<ConnectionRecord> clist = s.connections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                s.connections.put(binder, clist);
            }
            clist.add(c);
            ...
            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                s.lastActivity = SystemClock.uptimeMillis();
                //执行bringUpServiceLocked,bringUpServiceLocked调用到realStartServiceLocked;
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false) != null) {
                    return 0;
                }
            }


​            
            if (s.app != null && b.intent.received) {
                // Service is already running, so we can immediately
                // publish the connection.
                ...
                    //重复bind时会走这里，直接调用requestServiceBindingLocked
                    c.conn.connected(s.name, b.intent.binder);
                ...
                // If this is the first app connected back to this binding,
                // and the service had previously asked to be told when
                // rebound, then do so.
                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                    requestServiceBindingLocked(s, b.intent, callerFg, true);
                }
            } else if (!b.intent.requested) {
                requestServiceBindingLocked(s, b.intent, callerFg, false);
            
            }
        ...
        return 1;
    }

   

bringUpServiceLocked

     private final String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg, boolean whileRestarting) throws TransactionTooLargeException {
            ...
            if (!isolated) {
                app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
                if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                            + " app=" + app);
                if (app != null && app.thread != null) {
                    try {
                        app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                        realStartServiceLocked(r, app, execInFg);
                        return null;
                    } catch (TransactionTooLargeException e) {
                        throw e;
                    } catch (RemoteException e) {
                        Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                    }
                }
            }
            ...
            return null;
        }
 realStartServiceLocked

    private final void realStartServiceLocked(ServiceRecord r,
                ProcessRecord app, boolean execInFg) throws RemoteException {
    
            r.app = app;
           ...
            //3.调用ActivityThread的scheduleCreateService,创建一个Service
                app.thread.scheduleCreateService(r, r.serviceInfo,
                        mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                        app.repProcState);
                r.postNotification();
         ...
            //4. 等待3的service被创建完成bindService
            requestServiceBindingsLocked(r, execInFg);
            updateServiceClientActivitiesLocked(app, null, true);
        }

分析3过程
app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
                        
    //scheduleCreateService 发送一个消息
    public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
    
            sendMessage(H.CREATE_SERVICE, s);
    }
    //handleCreateService
    private void handleCreateService(CreateServiceData data) {
    ...
        初始化一个service ，并执行回调
        Service service = null;
        ...
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
            ...
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            service.onCreate();
            mServices.put(data.token, service);
            ...
            //serviceDoneExecuting的主要工作是当service启动完成，则移除service Timeout消息。
            ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            ...
    }
分析4过程：
requestServiceBindingsLocked(r, execInFg);

    private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
            ...
            //调用ActivityThread的scheduleBindService
            r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.repProcState);
            ...
    }

接下来的过程类似于 onCreate 的过程scheduleBindService -> handleMessage -> handleBindService 调用到ActiveServices.publishServiceLocked
    
    private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        ...
        //类似于之前的过程 最终调用到 ActiveServices 的 publishServiceLocked
            ActivityManagerNative.getDefault().publishService(
                                data.token, data.intent, binder);
        ...
    }
//取出在bindServiceLock中放到ServiceRecord中的ConnectionRecord 

    void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        for (int conni=r.connections.size()-1; conni>=0; conni--) {
             ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                 for (int i=0; i<clist.size(); i++) { ConnectionRecord c = clist.get(i); ... //回调到onServiceConnected，bindService结束 即是方法ConnectionRecord.ServiceDispatcher.InnerConnection.connected c.conn.connected(r.name, service); ... } } } 四、startService和bindService的流程的区别在于： 4.1在bringUpServiceLocked 中开启新的进程后，绕一圈再调用到realStartServiceLocked ActivityManagerService.startProcessLocked->ActivityThread.main->ActivityManagerNative.getDefault().attachApplicationLocked-> 
ActivityManagerService.attachApplicationLocked->ActiveServices.attachApplicationLocked->ActiveServices.realStartServiceLocked
4.2当realStartServiceLocked 中执行时，bindService会调用requestServiceBindingsLocked做绑定服务的下一步处理，这里创建的Service并不会回调onStartCommand，在realStartServiceLocked中对start和bind的操作做了区分
    //bringUpServiceLocked
      

     private final String bringUpServiceLocked(ServiceRecord r,int intentFlags, boolean execInFg, boolean whileRestarting) {
             if (app == null) {
                    if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                            "service", r.name, false, isolated, false)) == null) {
                       //...
                }  
            }

//开启新的进程

    public final class ActivityThread {
                public static void main(String[] args) {
                    Looper.prepareMainLooper();
                    ActivityThread thread = new ActivityThread();
                    thread.attach(false);
                    Looper.loop();
                    throw new RuntimeException("Main thread loop unexpectedly exited");
                }
                private void attach(boolean system) {
                    ...
                     mgr.attachApplication(mAppThread);
                    ...
            }

 


   ActivityManagerService.attachApplicationLocked

     private final boolean attachApplicationLocked(IApplicationThread thread,
                int pid) {
              //...
                thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                        profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                        app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(mConfiguration), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked());
            ...
            // Find any services that should be running in this process...
            //这里继续调用到ActiveService.attachApplicationLocked
                 didSomething |= mServices.attachApplicationLocked(app, processName);
          //...
            return true;
        }
 ActiveServices.attachApplicationLocked   

     boolean attachApplicationLocked(ProcessRecord proc, String processName)
            throws RemoteException {
            ...
            realStartServiceLocked(sr, proc, sr.createdFromFg);
            didSomething = true;
            ...
        return didSomething;
    }

ActiveServices.realStartServiceLocked

        private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        //...
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();
        app.services.add(r);
        bumpServiceExecutingLocked(r, execInFg, "create");
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked();
        boolean created = false;
        try {
            //通知创建Service并执行onCreate
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
          //...
        } finally {
            //...
        }
        //bindService 会执行这里,最终回调到方法onServiceConnected
          scheduleBindService -> handleMessage -> handleBindService 调用到ActiveServices.publishServiceLocked
        requestServiceBindingsLocked(r, execInFg);
        updateServiceClientActivitiesLocked(app, null, true);
        //这里不会加入到pendingStarts里面，所以不会执行onStartCommand
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }
        //startService 过程继续执行这里
        sendServiceArgsLocked(r, execInFg, true);
        //...
    }
ActiveServices.requestServiceBindingsLocked,bindService 会执行这里,最终回调到方法onServiceConnected
          scheduleBindService -> handleMessage -> handleBindService,调用到ActiveServices.publishServiceLocked     

    private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
            throws TransactionTooLargeException {
        for (int i=r.bindings.size()-1; i>=0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
                break;
            }
        }
    } 