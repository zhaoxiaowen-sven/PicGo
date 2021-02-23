# IntentService 源码分析

IntentService 可以用来顺序执行一些优先级较高的耗时后台任务，例如后台下载。

### 一、启动初始化
在onCreate中：创建HandlerThread，内部封装了 Looper，在这里创建HandlerThread并启动，所以启动 IntentService 不需要新建线程。
    
    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.
    
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();
    
        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
### 二、调用过程
每次启动Service时，都会调用一次onStartCommand，处理每个后台任务的intent，最后通过mServiceHandler发送消息去处理

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }


​    
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

### 三、处理后台任务
在ServiceHandler 的 handleMessage中，
1.onHandleIntent，处理外界启动时传递进来的intent，需要子类去复写，根据不同的intent做相应的处理
2.stopSelf(int startId)，停止服务，在停止服务时会去判断最近的启动的服务次数是否和startId相同，如果相同就会立即停止，否则不会停止服务，最终就是在执行完最后一个任务后才停止服务

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }
    
        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
    
    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);


​    
    public final void stopSelf(int startId) {
        if (mActivityManager == null) {
            return;
        }
        try {
            mActivityManager.stopServiceToken(
                    new ComponentName(this, mClassName), mToken, startId);
        } catch (RemoteException ex) {
        }
    }

### 总结：
1.IntentService 可用于执行可用来执行一些优先限比较高的后台任务耗时任务，当任务执行完成后会自动停止
2.IntentService 内部封装了HandlerThread和Handler，每次调用startService就会传递一个intent进来，相当于添加一个任务，执行时是按照启动的顺序执行

参考资料：
http://www.jianshu.com/p/332b6daf91f0
http://blog.csdn.net/mingli198611/article/details/8782772