[http://gityuan.com/2016/06/04/broadcast-receiver/](http://%E5%8F%82%E8%80%83%E5%8D%9A%E5%AE%A2)
[https://rawgit.com/prife/VirtualAppDoc/master/pngs/Broadcast.svg](http://%E6%B5%81%E7%A8%8B%E5%9B%BE)

# 一、概述
广播(Broadcast)机制用于进程/线程间通信，广播分为广播发送和广播接收两个过程，其中广播接收者BroadcastReceiver便是Android四大组件之一。

BroadcastReceiver分为两类：

    静态广播接收者：通过AndroidManifest.xml的标签来申明的BroadcastReceiver。
    动态广播接收者：通过AMS.registerReceiver()方式注册的BroadcastReceiver，动态注册更为灵活，可在不需要时通过unregisterReceiver()取消注册。

从广播发送方式可分为三类：

    普通广播：通过Context.sendBroadcast()发送，可并行处理
    有序广播：通过Context.sendOrderedBroadcast()发送，串行处理
    Sticky广播：通过Context.sendStickyBroadcast()发送


# 二、广播的注册过程
# 2.1 registerReceiver(BroadcastReceiver receiver, IntentFilter filter)
# 2.2 ContextWrapper.registerReceiver(receiver,filter)
# 2.3 ContextImpl.registerReceiver(receiver,filter)

    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }
    
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        //rd 的类型是LoadApk.ReceiverDispatcher.InnerReceiver的一个对象
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            //调用到AMS的registerReceiver方法
            return ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
        } catch (RemoteException e) {
            return null;
        }
    }


# step1： rd 的创建过程
ReceiverDispatcher(广播分发者)有一个内部类InnerReceiver，该类继承于IIntentReceiver.Stub。显然，这是一个Binder服务端，广播分发者通过rd.getIIntentReceiver()可获取该Binder服务端对象InnerReceiver，用于Binder IPC通信。

    frameworks/base/core/java/android/app/LoadedApk.java
    
    static final class ReceiverDispatcher {
    
        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;
    
            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                mStrongRef = strong ? rd : null;
            }
           ...
        }
    
        final IIntentReceiver.Stub mIIntentReceiver;
        final BroadcastReceiver mReceiver;
        ...
    
         ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                Handler activityThread, Instrumentation instrumentation,
                boolean registered) {
            if (activityThread == null) {
                throw new NullPointerException("Handler must not be null");
            }
    
            mIIntentReceiver = new InnerReceiver(this, !registered);
            mReceiver = receiver;
            mContext = context;
            mActivityThread = activityThread;
            mInstrumentation = instrumentation;
            mRegistered = registered;
            mLocation = new IntentReceiverLeaked(null);
            mLocation.fillInStackTrace();
        }
        ...
        IIntentReceiver getIIntentReceiver() {
            return mIIntentReceiver;
        }

# 2.4 在AMS中注册
其中mRegisteredReceivers记录着所有已注册的广播，以receiver IBinder为key, ReceiverList为value为HashMap。另外，这个过程涉及对象ReceiverList，BroadcastFilter，BroadcastRecord的创建。

在BroadcastQueue中有两个广播队列mParallelBroadcasts,mOrderedBroadcasts，数据类型都为ArrayList：

    mParallelBroadcasts:并行广播队列，可以立刻执行，而无需等待另一个广播运行完成，该队列只允许动态已注册的广播，从而避免发生同时拉起大量进程来执行广播，前台的和后台的广播分别位于独立的队列。
    mOrderedBroadcasts：有序广播队列，同一时间只允许执行一个广播，该队列顶部的广播便是活动广播，其他广播必须等待该广播结束才能运行，也是独立区别前台的和后台的广播。
    
    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
                IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
            synchronized (this) {
                if (callerApp != null && (callerApp.thread == null
                        || callerApp.thread.asBinder() != caller.asBinder())) {
                    return null;
                }
                //ReceiverList rl保存了相同InnerReceiver注册的广播接收者BroadcastFilter
                //mRegisteredReceivers的 final HashMap<IBinder, ReceiverList> mRegisteredReceivers
                //通过不同的InnerReceiver来区分不同的广播
                ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
                if (rl == null) {
                    rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                            userId, receiver);
                    if (rl.app != null) {
                        rl.app.receivers.add(rl);
                    } else {
                        try {
                            receiver.asBinder().linkToDeath(rl, 0);
                        } catch (RemoteException e) {
                            return sticky;
                        }
                        rl.linkedToDeath = true;
                    }
                    mRegisteredReceivers.put(receiver.asBinder(), rl);
                } 
                //描述正在注册的广播接收者
                BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                        permission, callingUid, userId);
                //将注册的广播接收者BroadcastFilter 加入到ReceiverList中
                rl.add(bf);
                if (!bf.debugCheck()) {
                    Slog.w(TAG, "==> For Dynamic broadcast");
                }
                //将广播接收者保存到内部的类成员变量mReceiverResolver中，所以BroadcastReceiver和InnerReceiver是成对存在的
                //final IntentResolver<BroadcastFilter, BroadcastFilter> mReceiverResolver
                mReceiverResolver.addFilter(bf);
    
                if (allSticky != null) {
                    ArrayList receivers = new ArrayList();
                    receivers.add(bf);
    
                    final int stickyCount = allSticky.size();
                    for (int i = 0; i < stickyCount; i++) {
                        Intent intent = allSticky.get(i);
                        BroadcastQueue queue = broadcastQueueForIntent(intent);
                        BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                                null, -1, -1, null, null, AppOpsManager.OP_NONE, null, receivers,
                                null, 0, null, null, false, true, true, -1);
                        queue.enqueueParallelBroadcastLocked(r);
                        queue.scheduleBroadcastsLocked();
                    }
                }
    
                return sticky;
            }
        }
# 小结

注册广播的过程，主要功能：
创建ReceiverList(接收者队列)，并添加到AMS.mRegisteredReceivers(已注册广播队列)；
创建BroadcastFilter(广播过滤者)，并添加到AMS.mReceiverResolver(接收者的解析人)；
当注册的是Sticky广播，则创建BroadcastRecord，并添加到BroadcastQueue的mParallelBroadcasts(并行广播队列)，注册后调用AMS来尽快处理该广播。

三、广播的发送过程
# 3.1 sendBroadcast(intent)

# 3.2 ContextWrapper.sendBroadcast(intent)

    frameworks/base/core/java/android/content/ContextWrapper.java
    
    public void sendBroadcast(Intent intent) {
            mBase.sendBroadcast(intent);
    }

# 3.3 ContextImpl.sendBroadcast

    frameworks/base/core/java/android/app/ContextImpl.java
    
    public void sendBroadcast(Intent intent) {
            warnIfCallingFromSystemProcess();
            String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
            try {
                intent.prepareToLeaveProcess();
                ActivityManagerNative.getDefault().broadcastIntent(
                        mMainThread.getApplicationThread(), intent, resolvedType, null,
                        Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                        getUserId());
            } catch (RemoteException e) {
                throw new RuntimeException("Failure from system", e);
            }
        }

# 3.4 ActivityManagerProxy.broadcastIntent

    frameworks/base/core/java/android/app/ActivityManagerNative.java
    
        public int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle map,
            String[] requiredPermissions, int appOp, Bundle options, boolean serialized,
            boolean sticky, int userId) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo != null ? resultTo.asBinder() : null);
        data.writeInt(resultCode);
        data.writeString(resultData);
        data.writeBundle(map);
        data.writeStringArray(requiredPermissions);
        data.writeInt(appOp);
        data.writeBundle(options);
        data.writeInt(serialized ? 1 : 0);
        data.writeInt(sticky ? 1 : 0);
        data.writeInt(userId);
        mRemote.transact(BROADCAST_INTENT_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        reply.recycle();
        data.recycle();
        return res;
    }

# 3.5 ActivityManagerService.broadcastIntent

    frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
    
    public final int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle resultExtras,
            String[] requiredPermissions, int appOp, Bundle options,
            boolean serialized, boolean sticky, int userId) {
        enforceNotIsolatedCaller("broadcastIntent");
        synchronized(this) {
            intent = verifyBroadcastLocked(intent);
    
            final ProcessRecord callerApp = getRecordForAppLocked(caller);
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            int res = broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null,
                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                    requiredPermissions, appOp, null, serialized, sticky,
                    callingPid, callingUid, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
# 3.6 ActivityManagerService.broadcastIntentLocked 分解为8个步骤解析


# step1: 设置广播flag

    添加flag=FLAG_EXCLUDE_STOPPED_PACKAGES，保证已停止app不会收到该广播；
    当系统还没有启动完成，则不允许启动新进程，，即只有动态注册receiver才能接受广播
    当非USER_ALL广播且当前用户并没有处于Running的情况下，除非是系统升级广播或者关机广播，否则直接返回。
    
    BroadcastReceiver还有其他flag，位于Intent.java常量:
        FLAG_RECEIVER_REGISTERED_ONLY //只允许已注册receiver接收广播
        FLAG_RECEIVER_REPLACE_PENDING //新广播会替代相同广播
        FLAG_RECEIVER_FOREGROUND //只允许前台receiver接收广播
        FLAG_RECEIVER_NO_ABORT //对于有序广播，先接收到的receiver无权抛弃广播
        FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT //Boot完成之前，只允许已注册receiver接收广播
        FLAG_RECEIVER_BOOT_UPGRADE //升级模式下，允许系统准备就绪前可以发送广播
    
    private final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle options,
            boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {
        intent = new Intent(intent);
    
        // By default broadcasts do not go to stopped apps.
        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
        
        // If we have not finished booting, don't allow this to launch new processes.
        if (!mProcessesReady && (intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0) {
            intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        }
        
         // Make sure that the user who is receiving this broadcast is running.
        // If not, we will just skip it. Make an exception for shutdown broadcasts
        // and upgrade steps.
    
        if (userId != UserHandle.USER_ALL && !isUserRunningLocked(userId, false)) {
            if ((callingUid != Process.SYSTEM_UID
                    || (intent.getFlags() & Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0)
                    && !Intent.ACTION_SHUTDOWN.equals(intent.getAction())) {
                Slog.w(TAG, "Skipping broadcast of " + intent
                        + ": user " + userId + " is stopped");
                return ActivityManager.BROADCAST_FAILED_USER_STOPPED;
            }
        }


# step2: 广播权限验证

对于callingAppId为SYSTEM_UID，PHONE_UID，SHELL_UID，BLUETOOTH_UID，NFC_UID之一或者callingUid == 0时都畅通无阻；
否则对于调用者进程为空并且不是persistent进程的情况下：
    1.当发送的是受保护广播mProtectedBroadcasts(只允许系统使用)，则抛出异常；
    2.当action为ACTION_APPWIDGET_CONFIGURE时，虽然不希望该应用发送这种广播，处于兼容性考虑，限制该广播只允许发送给自己，否则抛出异常。

    /*
         * Prevent non-system code (defined here to be non-persistent
         * processes) from sending protected broadcasts.
         */
        int callingAppId = UserHandle.getAppId(callingUid);
        if (callingAppId == Process.SYSTEM_UID || callingAppId == Process.PHONE_UID
            || callingAppId == Process.SHELL_UID || callingAppId == Process.BLUETOOTH_UID
            || callingAppId == Process.NFC_UID || callingUid == 0) {
            // Always okay.
        } else if (callerApp == null || !callerApp.persistent) {
            try {
                if (AppGlobals.getPackageManager().isProtectedBroadcast(
                        intent.getAction())) {
                    String msg = "Permission Denial: not allowed to send broadcast "
                            + intent.getAction() + " from pid="
                            + callingPid + ", uid=" + callingUid;
                    Slog.w(TAG, msg);
                    throw new SecurityException(msg);
                } else if (AppWidgetManager.ACTION_APPWIDGET_CONFIGURE.equals(intent.getAction())) {
                    // Special case for compatibility: we don't want apps to send this,
                    // but historically it has not been protected and apps may be using it
                    // to poke their own app widget.  So, instead of making it protected,
                    // just limit it to the caller.
                    if (callerApp == null) {
                        String msg = "Permission Denial: not allowed to send broadcast "
                                + intent.getAction() + " from unknown caller.";
                        Slog.w(TAG, msg);
                        throw new SecurityException(msg);
                    } else if (intent.getComponent() != null) {
                        // They are good enough to send to an explicit component...  verify
                        // it is being sent to the calling app.
                        if (!intent.getComponent().getPackageName().equals(
                                callerApp.info.packageName)) {
                            String msg = "Permission Denial: not allowed to send broadcast "
                                    + intent.getAction() + " to "
                                    + intent.getComponent().getPackageName() + " from "
                                    + callerApp.info.packageName;
                            Slog.w(TAG, msg);
                            throw new SecurityException(msg);
                        }
                    } else {
                        // Limit broadcast to their own package.
                        intent.setPackage(callerApp.info.packageName);
                    }
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Remote exception", e);
                return ActivityManager.BROADCAST_SUCCESS;
            }
        }
# step3: 处理系统相关广播

    这个过程代码较长，主要处于系统相关的广播，如下10个case：
    case Intent.ACTION_UID_REMOVED: //uid移除
    case Intent.ACTION_PACKAGE_REMOVED: //package移除，
    case Intent.ACTION_PACKAGE_CHANGED: //package改变
    case Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE: //外部设备不可用时，强制停止所有波及的应用并清空cache数据
    case Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE: //外部设备可用
    case Intent.ACTION_PACKAGE_ADDED: //增加package，处于兼容考虑
    
    case Intent.ACTION_TIMEZONE_CHANGED: //时区改变，通知所有运行中的进程
    case Intent.ACTION_TIME_CHANGED: //时间改变，通知所有运行中的进程
    case Intent.ACTION_CLEAR_DNS_CACHE: //dns缓存清空
    case Proxy.PROXY_CHANGE_ACTION: //网络代理改变
    
    final String action = intent.getAction();
    if (action != null) {
        switch (action) {
            case Intent.ACTION_UID_REMOVED:
                mBatteryStatsService.removeUid(uid);
                mAppOpsService.uidRemoved(uid);
                break;
            case Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE:
                String list[] = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
                if (list != null && list.length > 0) {
                    for (int i = 0; i < list.length; i++) {
                        forceStopPackageLocked(list[i], -1, false, true, true,
                                false, false, userId, "storage unmount");
                    }
                    mRecentTasks.cleanupLocked(UserHandle.USER_ALL);
                    sendPackageBroadcastLocked(
                        IApplicationThread.EXTERNAL_STORAGE_UNAVAILABLE, list,userId);
                }
                break;
            case Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE
                mRecentTasks.cleanupLocked(UserHandle.USER_ALL);
                break;
            case Intent.ACTION_PACKAGE_REMOVED:
            case Intent.ACTION_PACKAGE_CHANGED:
                Uri data = intent.getData();
                boolean removed = Intent.ACTION_PACKAGE_REMOVED.equals(action);
                boolean fullUninstall = removed && !intent.getBooleanExtra(Intent.EXTRA_REPLACING, false);
                final boolean killProcess = !intent.getBooleanExtra(Intent.EXTRA_DONT_KILL_APP, false);
                if (killProcess) {
                    forceStopPackageLocked(ssp, UserHandle.getAppId(
                            intent.getIntExtra(Intent.EXTRA_UID, -1)),
                            false, true, true, false, fullUninstall, userId,
                            removed ? "pkg removed" : "pkg changed");
                }
                if (removed) {
                    sendPackageBroadcastLocked(IApplicationThread.PACKAGE_REMOVED,new String[] {ssp}, userId);
                    if (fullUninstall) {
                        mAppOpsService.packageRemoved(intent.getIntExtra(Intent.EXTRA_UID, -1), ssp);
                        removeUriPermissionsForPackageLocked(ssp, userId, true);
                        removeTasksByPackageNameLocked(ssp, userId);
                        mBatteryStatsService.notePackageUninstalled(ssp);
                    }
                } else {
                    cleanupDisabledPackageComponentsLocked(ssp, userId, killProcess,
                            intent.getStringArrayExtra(Intent.EXTRA_CHANGED_COMPONENT_NAME_LIST));
                }
                break;
    
            case Intent.ACTION_PACKAGE_ADDED:
                Uri data = intent.getData();
                final boolean replacing =intent.getBooleanExtra(Intent.EXTRA_REPLACING, false);
                mCompatModePackages.handlePackageAddedLocked(ssp, replacing);
                ApplicationInfo ai = AppGlobals.getPackageManager().getApplicationInfo(ssp, 0, 0);
                break;
            case Intent.ACTION_TIMEZONE_CHANGED:
                mHandler.sendEmptyMessage(UPDATE_TIME_ZONE);
                break;
            case Intent.ACTION_TIME_CHANGED:
                final int is24Hour = intent.getBooleanExtra(Intent.EXTRA_TIME_PREF_24_HOUR_FORMAT, false) ? 1: 0;
                mHandler.sendMessage(mHandler.obtainMessage(UPDATE_TIME, is24Hour, 0));
                BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
                synchronized (stats) {
                    stats.noteCurrentTimeChangedLocked();
                }
                break;
            case Intent.ACTION_CLEAR_DNS_CACHE:
                mHandler.sendEmptyMessage(CLEAR_DNS_CACHE_MSG);
                break;
            case Proxy.PROXY_CHANGE_ACTION:
                ProxyInfo proxy = intent.getParcelableExtra(Proxy.EXTRA_PROXY_INFO);
                mHandler.sendMessage(mHandler.obtainMessage(UPDATE_HTTP_PROXY_MSG, proxy));
                break;
        }
    }
# step4：增加sticky广播
    这个过程主要是将sticky广播增加到list，并放入mStickyBroadcasts里面。
    
    if (sticky) {
        if (checkPermission(android.Manifest.permission.BROADCAST_STICKY,
                callingPid, callingUid)
                != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException("");
        }
        if (requiredPermissions != null && requiredPermissions.length > 0) {
            return ActivityManager.BROADCAST_STICKY_CANT_HAVE_PERMISSION;
        }
    
        if (intent.getComponent() != null) {
           //当sticky广播发送给指定组件，则throw Exception
        }
        if (userId != UserHandle.USER_ALL) {
           //当非USER_ALL广播跟USER_ALL广播出现冲突,则throw Exception
        }
    
        ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(userId);
        if (stickies == null) {
            stickies = new ArrayMap<>();
            mStickyBroadcasts.put(userId, stickies);
        }
        ArrayList<Intent> list = stickies.get(intent.getAction());
        if (list == null) {
            list = new ArrayList<>();
            stickies.put(intent.getAction(), list);
        }
        final int stickiesCount = list.size();
        int i;
        for (i = 0; i < stickiesCount; i++) {
            if (intent.filterEquals(list.get(i))) {
                //替换已存在的sticky intent
                list.set(i, new Intent(intent));
                break;
            }
        }
        //新的intent追加到list
        if (i >= stickiesCount) {
            list.add(new Intent(intent));
        }
    }
# step5：查询receivers和registeredReceivers

receivers：记录着匹配当前intent的所有静态注册广播接收者；
registeredReceivers：记录着匹配当前的所有动态注册的广播接收者。
其中，mReceiverResolver是AMS的成员变量，记录着已注册的广播接收者的resolver.

    int[] users;
    if (userId == UserHandle.USER_ALL) {
        users = mStartedUserArray; //广播给所有已启动用户
    } else {
        users = new int[] {userId}; //广播给指定用户
    }
    
    List receivers = null;
    List<BroadcastFilter> registeredReceivers = null;
    //找出所有能接收该广播的receivers
    if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
        //根据intent查找相应的receivers,查询静态注册的广播
        receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
    }
    if (intent.getComponent() == null) {
        if (userId == UserHandle.USER_ALL && callingUid == Process.SHELL_UID) {
            UserManagerService ums = getUserManagerLocked();
            for (int i = 0; i < users.length; i++) {
                //shell用户是否开启允许debug功能
                if (ums.hasUserRestriction(UserManager.DISALLOW_DEBUGGING_FEATURES, users[i])) {
                    continue;
                }
                // 查询动态注册的广播
                List<BroadcastFilter> registeredReceiversForUser =
                        mReceiverResolver.queryIntent(intent,
                                resolvedType, false, users[i]);
                if (registeredReceivers == null) {
                    registeredReceivers = registeredReceiversForUser;
                } else if (registeredReceiversForUser != null) {
                    registeredReceivers.addAll(registeredReceiversForUser);
                }
            }
        } else {
            // 查询动态注册的广播
            registeredReceivers = mReceiverResolver.queryIntent(intent,
                    resolvedType, false, userId);
        }
    }
    
    AMS.collectReceiverComponents：
    
    private List<ResolveInfo> collectReceiverComponents(Intent intent, String resolvedType,
        int callingUid, int[] users) {
    List<ResolveInfo> receivers = null;
    for (int user : users) {
        //调用PKMS.queryIntentReceivers，可获取AndroidManifest.xml声明的接收者信息
        List<ResolveInfo> newReceivers = AppGlobals.getPackageManager()
                .queryIntentReceivers(intent, resolvedType, STOCK_PM_FLAGS, user);
        if (receivers == null) {
            receivers = newReceivers;
        } else if (newReceivers != null) {
            ...
            //将所用户的receiver整合到receivers
        }
     }
    return receivers;
    }
# step6：处理并行广播
广播队列中有一个成员变量mParallelBroadcasts，类型为ArrayList，记录着所有的并行广播。
    
    //用于标识是否需要用新intent替换旧的intent。
    final boolean replacePending = (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;
    //处理并行广播
    int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
    if (!ordered && NR > 0) {
        final BroadcastQueue queue = broadcastQueueForIntent(intent);
        //创建BroadcastRecord对象
        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
                appOp, brOptions, registeredReceivers, resultTo, resultCode, resultData,
                resultExtras, ordered, sticky, false, userId);
    
        final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
        if (!replaced) {
            //将BroadcastRecord加入到并行广播队列
            queue.enqueueParallelBroadcastLocked(r);
            //处理广播【见小节4.1】
            queue.scheduleBroadcastsLocked();
        }
        registeredReceivers = null;
        NR = 0;
    }
# step7：合并registeredReceivers到receivers   

    int ir = 0;
    if (receivers != null) {
        //防止应用监听该广播，在安装时直接运行。
        String skipPackages[] = null;
        if (Intent.ACTION_PACKAGE_ADDED.equals(intent.getAction())
                || Intent.ACTION_PACKAGE_RESTARTED.equals(intent.getAction())
                || Intent.ACTION_PACKAGE_DATA_CLEARED.equals(intent.getAction())) {
            Uri data = intent.getData();
            if (data != null) {
                String pkgName = data.getSchemeSpecificPart();
                if (pkgName != null) {
                    skipPackages = new String[] { pkgName };
                }
            }
        } else if (Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE.equals(intent.getAction())) {
            skipPackages = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
        }
    
        //将skipPackages相关的广播接收者从receivers列表中移除
        if (skipPackages != null && (skipPackages.length > 0)) {
            for (String skipPackage : skipPackages) {
                if (skipPackage != null) {
                    int NT = receivers.size();
                    for (int it=0; it<NT; it++) { ResolveInfo curt = (ResolveInfo)receivers.get(it); if (curt.activityInfo.packageName.equals(skipPackage)) { receivers.remove(it); it--; NT--; } } } } } //前面part6有一个处理动态广播的过程，处理完后再执行将动态注册的registeredReceivers合并到receivers int NT = receivers != null ? receivers.size() : 0; int it = 0; ResolveInfo curt = null; BroadcastFilter curr = null; while (it < NT && ir < NR) { if (curt == null) { curt = (ResolveInfo)receivers.get(it); } if (curr == null) { curr = registeredReceivers.get(ir); } if (curr.getPriority()>= curt.priority) {
                receivers.add(it, curr);
                ir++;
                curr = null;
                it++;
                NT++;
            } else {
                it++;
                curt = null;
            }
        }
    }
    while (ir < NR) {
        if (receivers == null) {
            receivers = new ArrayList();
        }
        receivers.add(registeredReceivers.get(ir));
        ir++;
    }

# step8: 处理串行广播
广播队列中有一个成员变量mOrderedBroadcasts，类型为ArrayList，记录着所有的有序广播。

        if ((receivers != null && receivers.size() > 0)
            || resultTo != null) {
        BroadcastQueue queue = broadcastQueueForIntent(intent);
        //创建BroadcastRecord
        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                callerPackage, callingPid, callingUid, resolvedType,
                requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                resultData, resultExtras, ordered, sticky, false, userId);
    
        boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
        if (!replaced) {
            //将BroadcastRecord加入到有序广播队列
            queue.enqueueOrderedBroadcastLocked(r);
            //处理广播【见小节4.1】
            queue.scheduleBroadcastsLocked();
        }
    }

# 小结：
    注册广播的小节[2.4]阶段, 会处理Sticky广播;
    发送广播的[step 6]阶段, 会处理并行广播;
    发送广播的[step 8]阶段, 会处理串行广播;
上述3个处理过程都是通过调用scheduleBroadcastsLocked()方法来完成的,接下来再来看看这个方法.

# 四、 处理广播

在发送广播过程中会执行scheduleBroadcastsLocked方法来处理相关的广播
     
    base/services/core/java/com/android/server/am/BroadcastQueue
# 4.1 scheduleBroadcastsLocked
    public void scheduleBroadcastsLocked() {
            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                    + mQueueName + "]: current="
                    + mBroadcastsScheduled);
    
            if (mBroadcastsScheduled) {
                return;
            }
            mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
            mBroadcastsScheduled = true;
        }
# 4.2 handleMessage
调用到processNextBroadcast

    private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }
    
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                    processNextBroadcast(true);
                } break;
                ...
                }

# 4.3 processNextBroadcast
此处mService为AMS，整个流程还是比较长的，全程持有AMS锁，所以广播效率低的情况下，直接会严重影响这个手机的性能与流畅度，这里应该考虑细化同步锁的粒度。

    final void processNextBroadcast(boolean fromMsg) {
    synchronized(mService) {
        //step1: 处理并行广播
        //step2: 处理当前有序广播
        //step3: 获取下条有序广播
        //step4: 处理下条有序广播
        }
    }

# step1: 处理并行广播

    BroadcastRecord r;
    mService.updateCpuStats(); //更新CPU统计信息
    if (fromMsg)  mBroadcastsScheduled = false;
    
    while (mParallelBroadcasts.size() > 0) {
        r = mParallelBroadcasts.remove(0);
        r.dispatchTime = SystemClock.uptimeMillis();
        r.dispatchClockTime = System.currentTimeMillis();
        final int N = r.receivers.size();
        for (int i=0; i<N; i++) { Object target = r.receivers.get(i); //分发广播给已注册的receiver 【见小节4.3】 deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false); } addBroadcastToHistoryLocked(r);//将广播添加历史统计 } # step2: 处理当前有序广播 if (mPendingBroadcast != null) { boolean isDead; synchronized (mService.mPidsSelfLocked) { //从mPidsSelfLocked获取正在处理该广播进程，判断该进程是否死亡 ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid); isDead = proc == null || proc.crashing; } if (!isDead) { //正在处理广播的进程保持活跃状态，则继续等待其执行完成 return; } else { mPendingBroadcast.state = BroadcastRecord.IDLE; mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex; mPendingBroadcast = null; } } boolean looped = false; do { if (mOrderedBroadcasts.size() == 0) { //所有串行广播处理完成，则调度执行gc mService.scheduleAppGcsLocked(); if (looped) { mService.updateOomAdjLocked(); } return; } r = mOrderedBroadcasts.get(0); boolean forceReceive = false; //获取所有该广播所有的接收者 int numReceivers = (r.receivers != null) ? r.receivers.size() : 0; if (mService.mProcessesReady && r.dispatchTime> 0) {
            long now = SystemClock.uptimeMillis();
            if ((numReceivers > 0) &&
                    (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                //当广播处理时间超时，则强制结束这条广播
                broadcastTimeoutLocked(false);
                forceReceive = true;
                r.state = BroadcastRecord.IDLE;
            }
        }
    
        if (r.state != BroadcastRecord.IDLE) {
            return;
        }
    
        if (r.receivers == null || r.nextReceiver >= numReceivers
                || r.resultAbort || forceReceive) {
            if (r.resultTo != null) {
                //处理广播消息消息，调用到onReceive()
                performReceiveLocked(r.callerApp, r.resultTo,
                    new Intent(r.intent), r.resultCode,
                    r.resultData, r.resultExtras, false, false, r.userId);
                r.resultTo = null;
            }
            //取消BROADCAST_TIMEOUT_MSG消息
            cancelBroadcastTimeoutLocked();
    
            addBroadcastToHistoryLocked(r);
            mOrderedBroadcasts.remove(0);
            r = null;
            looped = true;
            continue;
        }
    } while (r == null);

# step3: 获取下条有序广播
mTimeoutPeriod，对于前台广播则为10s，对于后台广播则为60s。广播超时为2*mTimeoutPeriod*numReceivers，接收者个数numReceivers越多则广播超时总时长越大。

    //获取下一个receiver的index
    int recIdx = r.nextReceiver++;
    
    r.receiverTime = SystemClock.uptimeMillis();
    if (recIdx == 0) {
        r.dispatchTime = r.receiverTime;
        r.dispatchClockTime = System.currentTimeMillis();
    }
    if (!mPendingBroadcastTimeoutMessage) {
        long timeoutTime = r.receiverTime + mTimeoutPeriod;
        //设置广播超时时间，发送BROADCAST_TIMEOUT_MSG
        setBroadcastTimeoutLocked(timeoutTime);
    }
    
    final BroadcastOptions brOptions = r.options;
    //获取下一个广播接收者
    final Object nextReceiver = r.receivers.get(recIdx);
    
    if (nextReceiver instanceof BroadcastFilter) {
        //对于动态注册的广播接收者，deliverToRegisteredReceiverLocked处理广播
        BroadcastFilter filter = (BroadcastFilter)nextReceiver;
        deliverToRegisteredReceiverLocked(r, filter, r.ordered);
        if (r.receiver == null || !r.ordered) {
            r.state = BroadcastRecord.IDLE;
            scheduleBroadcastsLocked();
        } else {
            ...
        }
        return;
    }
    
    //对于静态注册的广播接收者
    ResolveInfo info = (ResolveInfo)nextReceiver;
    ComponentName component = new ComponentName(
            info.activityInfo.applicationInfo.packageName,
            info.activityInfo.name);
    ...
    //执行各种权限检测，此处省略，当权限不满足时skip=true
    
    if (skip) {
        r.receiver = null;
        r.curFilter = null;
        r.state = BroadcastRecord.IDLE;
        scheduleBroadcastsLocked();
        return;
    }
    
    r.state = BroadcastRecord.APP_RECEIVE;
    String targetProcess = info.activityInfo.processName;
    r.curComponent = component;
    final int receiverUid = info.activityInfo.applicationInfo.uid;
    if (r.callingUid != Process.SYSTEM_UID && isSingleton
            && mService.isValidSingletonCall(r.callingUid, receiverUid)) {
        info.activityInfo = mService.getActivityInfoForUser(info.activityInfo, 0);
    }
    r.curReceiver = info.activityInfo;
    ...
    
    //Broadcast正在执行中，stopped状态设置成false
    AppGlobals.getPackageManager().setPackageStoppedState(
            r.curComponent.getPackageName(), false, UserHandle.getUserId(r.callingUid));
# step4: 处理下条有序广播

如果是动态广播接收者，则调用deliverToRegisteredReceiverLocked处理；
如果是静态广播接收者，且对应进程已经创建，则调用processCurBroadcastLocked处理；
如果是静态广播接收者，且对应进程尚未创建，则调用startProcessLocked创建进程。

    //该receiver所对应的进程已经运行，则直接处理
        ProcessRecord app = mService.getProcessRecordLocked(targetProcess,
                info.activityInfo.applicationInfo.uid, false);
        if (app != null && app.thread != null) {
            try {
                app.addPackage(info.activityInfo.packageName,
                        info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
                processCurBroadcastLocked(r, app);
                return;
            } catch (RemoteException e) {
            } catch (RuntimeException e) {
                finishReceiverLocked(r, r.resultCode, r.resultData, r.resultExtras, r.resultAbort, false);
                scheduleBroadcastsLocked();
                r.state = BroadcastRecord.IDLE; //启动receiver失败则重置状态
                return;
            }
        }
        
        //该receiver所对应的进程尚未启动，则创建该进程
        if ((r.curApp=mService.startProcessLocked(targetProcess,
                info.activityInfo.applicationInfo, true,
                r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                "broadcast", r.curComponent,
                (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                        == null) {
            //创建失败，则结束该receiver
            finishReceiverLocked(r, r.resultCode, r.resultData,
                    r.resultExtras, r.resultAbort, false);
            scheduleBroadcastsLocked();
            r.state = BroadcastRecord.IDLE;
            return;
        }
        mPendingBroadcast = r;
        mPendingBroadcastRecvIndex = recIdx;

# 4.3 deliverToRegisteredReceiverLocked
    private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
            BroadcastFilter filter, boolean ordered) {
            ...
            //检查发送者是否有BroadcastFilter所需权限
            //以及接收者是否有发送者所需的权限等等
            //当权限不满足要求，则skip=true。
        
            if (!skip) {
                //并行广播ordered = false，只有串行广播才进入该分支
                if (ordered) {
                    r.receiver = filter.receiverList.receiver.asBinder();
                    r.curFilter = filter;
                    filter.receiverList.curBroadcast = r;
                    r.state = BroadcastRecord.CALL_IN_RECEIVE;
                    if (filter.receiverList.app != null) {
                        r.curApp = filter.receiverList.app;
                        filter.receiverList.app.curReceiver = r;
                        mService.updateOomAdjLocked(r.curApp);
                    }
                }
                // 处理广播【见小节4.4】
                performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                        new Intent(r.intent), r.resultCode, r.resultData,
                        r.resultExtras, r.ordered, r.initialSticky, r.userId);
                if (ordered) {
                    r.state = BroadcastRecord.CALL_DONE_RECEIVE;
                }
                ...
            }
        }
# 4.4 performReceiveLocked
    private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
        Intent intent, int resultCode, String data, Bundle extras,
        boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
    //通过binder异步机制，向receiver发送intent
    if (app != null) {
        if (app.thread != null) {
            //调用ApplicationThreadProxy类对应的方法 【4.5】
            app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                    data, extras, ordered, sticky, sendingUser, app.repProcState);
        } else {
            //应用进程死亡，则Recevier并不存在
            throw new RemoteException("app.thread must not be null");
        }
    } else {
        //调用者进程为空，则执行该分支
        receiver.performReceive(intent, resultCode, data, extras, ordered,
                sticky, sendingUser);
    }
}
# 4.5 ATP.scheduleRegisteredReceiver
ATP位于system_server进程，是Binder Bp端通过Binder驱动向Binder Bn端发送消息, ATP所对应的Bn端位于发送广播调用端所在进程的ApplicationThread，即进入AT.scheduleRegisteredReceiver， 接下来说明该方

    public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
            int resultCode, String dataStr, Bundle extras, boolean ordered,
            boolean sticky, int sendingUser, int processState) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(receiver.asBinder());
        intent.writeToParcel(data, 0);
        data.writeInt(resultCode);
        data.writeString(dataStr);
        data.writeBundle(extras);
        data.writeInt(ordered ? 1 : 0);
        data.writeInt(sticky ? 1 : 0);
        data.writeInt(sendingUser);
        data.writeInt(processState);
    
        //command=SCHEDULE_REGISTERED_RECEIVER_TRANSACTION
        mRemote.transact(SCHEDULE_REGISTERED_RECEIVER_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }

# 4.6 scheduleRegisteredReceiver
IPC过程 最终调用到ActivityThread.scheduleRegisteredReceiver,此处receiver是注册广播时创建的，见小节[2.3]，可知该receiver=LoadedApk.ReceiverDispatcher.InnerReceiver。

    public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
            int resultCode, String dataStr, Bundle extras, boolean ordered,
            boolean sticky, int sendingUser, int processState) throws RemoteException {
        //更新虚拟机进程状态
        updateProcessState(processState, false);
        //【见小节4.7】
        receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                sticky, sendingUser);
    }
# 4.7 InnerReceiver.performReceive
    base/core/java/android/app/LoadedApk.java
    public void performReceive(Intent intent, int resultCode, String data,
            Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
        LoadedApk.ReceiverDispatcher rd = mDispatcher.get();
        if (rd != null) {
            //【4.8】
            rd.performReceive(intent, resultCode, data, extras, ordered, sticky, sendingUser);
        } else {
           ...
        }
    }
# 4.8 ReceiverDispatcher.performReceive
其中Args继承于BroadcastReceiver.PendingResult，实现了接口Runnable。这里mActivityThread.post(args) 消息机制，关于Handler消息机制，见Android消息机制1-Handler(Java层)，把消息放入MessageQueue，再调用Args的run()方法。

    public void performReceive(Intent intent, int resultCode, String data,
            Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
        Args args = new Args(intent, resultCode, data, extras, ordered,
                sticky, sendingUser);
        //通过handler消息机制发送args.
        if (!mActivityThread.post(args)) {
            if (mRegistered && ordered) {
                IActivityManager mgr = ActivityManagerNative.getDefault();
                args.sendFinished(mgr);
            }
        }
    }
# 4.9 Args.run
最终调用BroadcastReceiver具体实现类的onReceive()方法，至此广播的处理过程结束,后续是AMS的一些处理过程
    
    public final class LoadedApk {
      static final class ReceiverDispatcher {
        final class Args extends BroadcastReceiver.PendingResult implements Runnable {
            public void run() {
                final BroadcastReceiver receiver = mReceiver;
                final boolean ordered = mOrdered;
    
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                final Intent intent = mCurIntent;
                mCurIntent = null;
    
                if (receiver == null || mForgotten) {
                    if (mRegistered && ordered) {
                        sendFinished(mgr);
                    }
                    return;
                }
    
                try {
                    //获取mReceiver的类加载器
                    ClassLoader cl =  mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    setExtrasClassLoader(cl);
                    receiver.setPendingResult(this);
                    //回调广播onReceive方法
                    receiver.onReceive(mContext, intent);
                } catch (Exception e) {
                    ...
                }
                //调用到BroadcastReceiver.finishReceiver
                if (receiver.getPendingResult() != null) {
                    finish();
                }
            }
          }
        }


​        
# 4.10 PendingResult.finish
此处AMP.finishReceiver，经过binder调用，进入AMS.finishReceiver方法,
    
    base/core/java/android/content/BroadcastReceiver.java
    
    public final void finish() {
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        sendFinished(mgr);
        ...
    }
    
    public void sendFinished(IActivityManager am) {
        synchronized (this) {
            try {
                if (mResultExtras != null) {
                    mResultExtras.setAllowFds(false);
                }
                if (mOrderedHint) {
                    //串行广播
                    am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras,
                            mAbortBroadcast, mFlags);
                } else {
                    //并行广播
                    am.finishReceiver(mToken, 0, null, null, false, mFlags);
                }
            } catch (RemoteException ex) {
            }
        }
    }

# 4.11 AMS.finishReceiver

    public void finishReceiver(IBinder who, int resultCode, String resultData,
            Bundle resultExtras, boolean resultAbort, int flags) {
        ...
        final long origId = Binder.clearCallingIdentity();
        try {
            boolean doNext = false;
            BroadcastRecord r;
    
            synchronized(this) {
                BroadcastQueue queue = (flags & Intent.FLAG_RECEIVER_FOREGROUND) != 0
                        ? mFgBroadcastQueue : mBgBroadcastQueue;
                r = queue.getMatchingOrderedReceiver(who);
                if (r != null) {
                    doNext = r.queue.finishReceiverLocked(r, resultCode,
                        resultData, resultExtras, resultAbort, true);
                }
            }
    
            if (doNext) {
                //处理下一条广播
                r.queue.processNextBroadcast(false);
            }
            trimApplications();
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }