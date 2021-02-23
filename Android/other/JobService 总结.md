Google在Android 5.0中引入JobScheduler来执行一些需要满足特定条件的后台任务，主要解决了某个任务需要在某种或者某几种条件满足之后触发的需求。这些条件包括网络状态，电池充电等。在JobScheduler出现之前，我们只能通过AlarmManager定时唤起应用或者监听广播的方式来做这件事，而一些系统广播是无法被静态注册的。这些做法效率很低，使应用进程不断的启动，甚至需要保活，浪费系统资源。很显然，JobScheduler很好的解决了这个问题。本文主要介绍下JobService的用法和一些常见的问题，并且分析下JobSchedule.schedule到JobService.onStartJob的执行过程。 

##一、基本用法
###1.新建MyJobService继承JobService

    public class MyJobService extends JobService {
        @Override public boolean onStartJob(JobParameters params) {
            doSampleJob(params); 
            return true;
        }
    
        @Override public boolean onStopJob(JobParameters params) {
            return false;
        }
    
        public void doSampleJob(JobParameters params) {
            // Do some heavy operation
            ...... 
            // At the end inform job manager the status of the job.
            jobFinished(params, false);
        }
    }
    
    注意：JobService要声明权限
    <service
        android:name=".MyJobService"
        android:permission="android.permission.BIND_JOB_SERVICE">

###2.通过JobInfo.Builder，创建JobInfo设置JobService执行的参数

    ComponentName serviceName = new ComponentName("com.example.aven.jobdemo",
           "com.example.aven.jobdemo.MyJobService");
    JobInfo jobInfo = new JobInfo.Builder(JOB_ID, serviceName)
           //.setMinimumLatency(3000)
           .setPeriodic(15 * 60 * 1000)
           .build();

###3.使用JobSchduler开启服务

    JobScheduler jobScheduler = (JobScheduler)this.getSystemService(Context.JOB_SCHEDULER_SERVICE);
    jobScheduler.schedule(jobInfo);

##二、API
使用JobService 主要会涉及到的3个类，JobScheduler，JobService，JobInfo.Builder(JobInfo)，重要的api和方法摘要如下。

###1.JobScheduler

    schedule(JobInfo job) 开启任务
    
    cancel(int jobId) 取消任务


​    
###2.JobService 
​    
1.onStartjob（jobparameters params）
如果返回值是true，那么表示系统执行的是耗时任务，当给定的任务完成时，需要手动调用jobFinished来停止该任务。
​    
2.onStopJob(JobParameters params)
系统停止服务时会调用。返回值true，表示任务会在满足setBackoffCriteria条件下重复执行。
​    
3.jobFinished(JobParameters params, boolean needsRescheduled) 
表示任务执行完成，两个参数值一个是JobParameters传递到JobService类，一个布尔值让系统知道是否需要根据工作的最初要求重新编排工作，类似于onStopJob的返回值。
​    
###3.JobInfo.Builder
​    
1.setMinimumLatency(long minLatencyMillis)：
这会使你的工作不启动直到规定的毫秒数已经过去了。这是与setPeriodic(long time)不兼容的，并且如果同时使用这两个函数将会导致抛出异常。

2.setOverrideDeadline(long maxExecutionDelayMillis)：
这将设置你的工作期限。即使是无法满足其他要求，你的任务将约在规定的时间已经过去时开始执行。类似于setMinimumLatency(long time)，这个函数是与 setPeriodic(long time) 互相排斥的，并且如果同时使用这两个函数，将会导致抛出异常。

3.setPersisted(boolean isPersisted)：这个函数告诉系统，在设备重新启动后，你的任务是否应该继续存在，只有能够监听开机广播的应用才可设置。
    
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <action android:name="android.intent.action.BOOT_COMPLETED" />

4.setPeriodic(long intervalMillis) 
设置任务重复执行的周期。
5.setPeriodic(long intervalMillis, long flexMillis)
同上 API 24 新增，增加限制intervalMillis >= 15min 和 flexMillis >= 5min.
6.setRequiredNetworkType(int networkType)：
只有在设备处于一种特定的网络中时，它才启动。它的默认值是JobInfo.NETWORK_TYPE_NONE，这就意味着，无论是否有网络连接，该任务均可以运行。另外两个可用的类型是JobInfo.NETWORK_TYPE_ANY，这需要某种类型的网络连接可用，工作才可以运行；以及JobInfo.NETWORK_TYPE_UNMETERED，这就要求设备在非蜂窝网络中。
    
7.setRequiresCharging(boolean requiresCharging)：使用这个函数会告诉你的应用程序，除非设备开始充电，否则工作不会启动。
    
8.setRequiresDeviceIdle(boolean requiresDeviceIdle)：
这会告知你的工作不会启动，除非用户不使用他们的设备，并且他们已经有一段时间没有使用它。
    
9.setBackoffCriteria(long initialBackoffMillis, int backoffPolicy) 第一个参数时第一次尝试重试的等待间隔，单位为毫秒。
第二个参数为 执行失败重试的策略，其实就是重试的时间间隔
BACKOFF_POLICY_LINEAR ：retry_time(current_time, num_failures) = current_time + initial_backoff_millis * num_failures
BACKOFF_POLICY_EXPONENTIAL ：retry_time(current_time, num_failures) = current_time + initial_backoff_millis * 2 ^ (num_failures - 1), num_failures >= 1

注意：setRequiredNetworkType(int networkType)、setRequiresCharging(boolean requireCharging)和setRequiresDeviceIdle(boolean requireIdle)可能会导致你的工作永远不启动，除非setOverrideDeadline(long time)还设置允许即使不符合条件的情况下，你的工作也可以运行。
    
###4.Tips：

1.Android7.0 以上机器执行任务周期小于15分钟定时任务的方法(待验证？)
    
    JobInfo jobInfo;
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
      // 同时需要设置 jobFinished needsRescheduled = true
      jobInfo = new JobInfo.Builder(JOB_ID, serviceName)
          .setMinimumLatency(REFRESH_INTERVAL)
          .setExtras(bundle).build();
    } else {
      jobInfo = new JobInfo.Builder(JOB_ID, serviceName)
          .setPeriodic(REFRESH_INTERVAL)
          .setExtras(bundle).build();
    }

2.判断任务是否正常执行

    int ret = jobScheduler.schedule(jobInfo);
    if (ret == JobScheduler.RESULT_SUCCESS) {
         Log.d(TAG, "Job scheduled successfully");
     } else {
         Log.d(TAG, "Job scheduling failed");
     }    

3.设置Jobfinished（true）时，会一直重复无论是否设置setPeriodic


##三、调试
除了在代码中加log外，JobScheduler 可以通过以下命令方便的dump出JobScheduler执行的信息:

    adb shell dumpsys jobscheduler

主要包括2部分：1.Job任务的属性 2.Job的执行记录

    JOB #u0a191/1001: 39fd118 com.example.aven.jobdemo/.MyJobService
       u0a191 tag=*job*/com.example.aven.jobdemo/.MyJobService
       Source: uid=u0a191 user=0 pkg=com.example.aven.jobdemo
       JobInfo:
         Service: com.example.aven.jobdemo/.MyJobService
         PERIODIC: interval=+15m0s0ms flex=+15m0s0ms
         Requires: charging=false deviceIdle=false
         Network type: 2
         Backoff: policy=0 initial=+5s0ms
         Has early constraint
         Has late constraint
       Required constraints: TIMING_DELAY DEADLINE UNMETERED
       Satisfied constraints: CONNECTIVITY UNMETERED NOT_ROAMING APP_NOT_IDLE DEVICE_NOT_DOZING
       Unsatisfied constraints: TIMING_DELAY DEADLINE
       Earliest run time: 04:12
       Latest run time: 19:12
       Ready: false (job=false pending=false active=false user=true)

adb shell dumpsys jobscheduler | grep com.example.aven.jobdemo

    JOB #u0a191/1001: d5ed9c8 com.example.aven.jobdemo/.MyJobService
      u0a191 tag=*job*/com.example.aven.jobdemo/.MyJobService
      Source: uid=u0a191 user=0 pkg=com.example.aven.jobdemo
        Service: com.example.aven.jobdemo/.MyJobService
    #u0a191/1001 from u0a191: com.example.aven.jobdemo RUNNABLE
    #u0a191/1001 from u0a191: com.example.aven.jobdemo RUNNABLE
    u0a191 / com.example.aven.jobdemo: 3x pending 1% 3x active
    u0a191 / com.example.aven.jobdemo: 2x pending 1x active 1x active-top
    u0a191 / com.example.aven.jobdemo: 3x pending 2x active 1x active-top
       -1h47m12s726ms START: u0a191 com.example.aven.jobdemo/.MyJobService
       -1h47m09s669ms  STOP: u0a191 com.example.aven.jobdemo/.MyJobService
       -1h32m17s690ms START: u0a191 com.example.aven.jobdemo/.MyJobService
       -1h32m14s641ms  STOP: u0a191 com.example.aven.jobdemo/.MyJobService
       -1h17m17s681ms START: u0a191 com.example.aven.jobdemo/.MyJobService
       -1h17m14s648ms  STOP: u0a191 com.example.aven.jobdemo/.MyJobService
       -1h02m17s704ms START: u0a191 com.example.aven.jobdemo/.MyJobService
       -1h02m14s667ms  STOP: u0a191 com.example.aven.jobdemo/.MyJobService
         -47m17s715ms START: u0a191 com.example.aven.jobdemo/.MyJobService
         -47m14s665ms  STOP: u0a191 com.example.aven.jobdemo/.MyJobService
         -27m33s752ms START: u0a191 com.example.aven.jobdemo/.MyJobService
         -27m30s648ms  STOP: u0a191 com.example.aven.jobdemo/.MyJobService
         -17m17s700ms START: u0a191 com.example.aven.jobdemo/.MyJobService
         -17m14s652ms  STOP: u0a191 com.example.aven.jobdemo/.MyJobService
          -2m17s721ms START: u0a191 com.example.aven.jobdemo/.MyJobService
          -2m14s687ms  STOP: u0a191 com.example.aven.jobdemo/.MyJobService

7.0（Android N版本）以上设置循环定时任务的方法：
1.period>=15min
2.jobfinished false
3.onStop 返回true

对于非persist的job杀掉应用后，Job会被取消掉，persist的不会，可以通过dump 命令查看。

##四、原理分析

## 1.获取系统服务，得到 Binder -> JobSchedulerService.JobSchedulerStub

    JobScheduler jobScheduler = (JobScheduler)this.getSystemService(Context.JOB_SCHEDULER_SERVICE);

###1./frameworks/base/core/java/android/app/JobSchedulerImpl.java
    public class JobSchedulerImpl extends JobScheduler {
        IJobScheduler mBinder;
    
        /* package */ JobSchedulerImpl(IJobScheduler binder) {
            mBinder = binder;
        }
    
        @Override
        public int schedule(JobInfo job) {
            try {
                return mBinder.schedule(job);
            } catch (RemoteException e) {
                return JobScheduler.RESULT_FAILURE;
            }
        }
        ...
    }

###2./frameworks/base/core/java/android/app/job/IJobScheduler.aidl

    interface IJobScheduler {
        int schedule(in JobInfo job);
        int enqueue(in JobInfo job, in JobWorkItem work);
        int scheduleAsPackage(in JobInfo job, String packageName, int userId, String tag);
        void cancel(int jobId);
        void cancelAll();
        List<JobInfo> getAllPendingJobs();
        JobInfo getPendingJob(int jobId);
    }

###3./frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java
    public final class JobSchedulerService{
        ...
        /**
         * Binder stub trampoline implementation
         */
        final class JobSchedulerStub extends IJobScheduler.Stub {
            ...
            @Override
            public int schedule(JobInfo job) throws RemoteException {
                ...
                try {
                    return JobSchedulerService.this.schedule(job, uid);
                } finally {
                    Binder.restoreCallingIdentity(ident);
                }
            }
        }
        ...
    }

## 2.启动定时任务：

    JobSchedulerService.schedule(jobInfo);
    
    1.将JobInfo转化成JobStatus
    2.校验任务：(1)package是否允许开启scheduleService getAppStartMode
                (2)package（uid）的任务有没有超过100 
                (3)是不是重复的任务
    3.跟踪任务

### 2.1./frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java
    public final class JobSchedulerService{
        ...
        /**
         * Entry point from client to schedule the provided job.
         * This cancels the job if it's already been scheduled, and replaces it with the one provided.
         */
        public int schedule(JobInfo job, int uId) {
            return scheduleAsPackage(job, uId, null, -1, null);
        }
        
        public int scheduleAsPackage(JobInfo job, int uId, String packageName, int userId,
                String tag) {
            JobStatus jobStatus = JobStatus.createFromJobInfo(job, uId, packageName, userId, tag);
            JobStatus toCancel;
            synchronized (mLock) {
                // Jobs on behalf of others don't apply to the per-app job cap
                if (ENFORCE_MAX_JOBS && packageName == null) {
                    if (mJobs.countJobsForUid(uId) > MAX_JOBS_PER_APP) {
                        Slog.w(TAG, "Too many jobs for uid " + uId);
                        throw new IllegalStateException("Apps may not schedule more than "
                                    + MAX_JOBS_PER_APP + " distinct jobs");
                    }
                }
                toCancel = mJobs.getJobByUidAndJobId(uId, job.getId());
                if (toCancel != null) {
                    cancelJobImpl(toCancel, jobStatus);
                }
                startTrackingJob(jobStatus, toCancel);
            }
            mHandler.obtainMessage(MSG_CHECK_JOB).sendToTarget();
            return JobScheduler.RESULT_SUCCESS;
        }
    }

## 2.2 JobSchedulerService.JobHandler.handMessage

### 2.2.1 maybeQueueReadyJobsForExecutionLockedH
          (1) 先将所有JobStatus加入runnableJobs队列;
          (2) 再将runnableJobs中满足触发条件的JobStatus加入到mPendingJobs队列;

### 2.2.2 maybeRunPendingJobsH -> assignJobsToContextsLocked 
          处理mPendingJobs列队中所有的Job.
    
    private class JobHandler extends Handler {
    ...
    @Override
    public void handleMessage(Message message) {
        synchronized (mLock) {
        ...
        switch (message.what) {
            case MSG_CHECK_JOB:
                synchronized (mLock) {
                    if (mReportedActive) {
                        // if jobs are currently being run, queue all ready jobs for execution.
                        queueReadyJobsForExecutionLockedH();
                    } else {
                        // Check the list of jobs and run some of them if we feel inclined.
                        maybeQueueReadyJobsForExecutionLockedH();
                    }
                }
                break;
            ...
        }
        maybeRunPendingJobsH();
        ...
    }
    
    private void maybeRunPendingJobsH() {
        synchronized (mLock) {
            if (DEBUG) {
                Slog.d(TAG, "pending queue: " + mPendingJobs.size() + " jobs.");
            }
            assignJobsToContextsLocked();
            reportActive();
        }
    }
    
    private void assignJobsToContextsLocked() {
        ...
        for (int i=0; i<MAX_JOB_CONTEXTS_COUNT; i++) {
                boolean preservePreferredUid = false;
                ...
                        if (!mActiveServices.get(i).executeRunnableJob(pendingJob)) {
                            Slog.d(TAG, "Error executing " + pendingJob);
                        }
                ...
            }
        ...
    }


​    
### 2.2.3 JobServiceContext.executeRunnableJob 
bindService的方式来拉起的进程，当服务启动后回调到onServiceConnected
    
    boolean executeRunnableJob(JobStatus job) {
        ...
        boolean binding = mContext.bindServiceAsUser(intent, this,
                        Context.BIND_AUTO_CREATE | Context.BIND_NOT_FOREGROUND,
                        new UserHandle(job.getUserId()));   
        ...
        }
    }
    
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        ...
        this.service = IJobService.Stub.asInterface(service);
        ...
        mCallbackHandler.obtainMessage(MSG_SERVICE_BOUND).sendToTarget();
    }

### 2.2.4 JobServiceContext.mCallbackHandler.handleMessage 

    /** Start the job on the service. */
        private void handleServiceBoundH() {
        ...
            try {
                mVerb = VERB_STARTING;
                scheduleOpTimeOut();
                service.startJob(mParams);
            } catch (RemoteException e) {
                Slog.e(TAG, "Error sending onStart message to '" +
                        mRunningJob.getServiceComponent().getShortClassName() + "' ", e);
            }
        }
    }

### 2.3 JobInterface.startJob 

    static final class JobInterface extends IJobService.Stub {
        final WeakReference<JobService> mService;
        JobInterface(JobService service) {
            mService = new WeakReference<>(service);
        }
        @Override
        public void startJob(JobParameters jobParams) throws RemoteException {
            JobService service = mService.get();
            if (service != null) {
                service.ensureHandler();
                Message m = Message.obtain(service.mHandler, MSG_EXECUTE_JOB, jobParams);
                m.sendToTarget();
            }
        }
        @Override
        public void stopJob(JobParameters jobParams) throws RemoteException {
            JobService service = mService.get();
            if (service != null) {
                service.ensureHandler();
                Message m = Message.obtain(service.mHandler, MSG_STOP_JOB, jobParams);
                m.sendToTarget();
            }
        }
    }

### 2.4 JobService.JobHandler.handlMessage

    class JobHandler extends Handler {
       ...
        @Override
        public void handleMessage(Message msg) {
            final JobParameters params = (JobParameters) msg.obj;
            switch (msg.what) {
                case MSG_EXECUTE_JOB:
                    try {
                        boolean workOngoing = JobService.this.onStartJob(params);
                        ackStartMessage(params, workOngoing);
                    } catch (Exception e) {
                        Log.e(TAG, "Error while executing job: " + params.getJobId());
                        throw new RuntimeException(e);
                    }
                    break;
                    ....
                }
            }
        }
    }

### 2.5 小结    
整个过程涉及两次跨进程的调用, 第一次是从app通过IJobScheduler调用到framework的JobScheduleService.schedule方法。 第二次则是framework 采用bindService方式通过IJobService接口，然后调用JobService的onStartJob。
![job_0](https://leanote.com/api/file/getImage?fileId=5a573518ab64415fdc001540)

# 3 JobStore 和 StateController
### 3.1 JobInfo, JobStatus，JobStore
保存Job信息，设置persistJob的会写入到配置文件/data/system/job/jobs.xml中，在开机启动时读取该文件，即使重启手机任务也可执行。
![job_1](https://leanote.com/api/file/getImage?fileId=5a585894ab64413e0d001645)

### 3.2 StateController 
监听网络等的变化，触发任务的执行
![job_2](https://leanote.com/api/file/getImage?fileId=5a55c4c3ab64417f83001ee6)

    /** List of controllers that will notify this service of updates to jobs. */
    List<StateController> mControllers;
    
    public JobSchedulerService(Context context) {
        super(context);
        mHandler = new JobHandler(context.getMainLooper());
        mConstants = new Constants(mHandler);
        mJobSchedulerStub = new JobSchedulerStub();
        mJobs = JobStore.initAndGet(this);
        // Create the controllers.
        mControllers = new ArrayList<StateController>();
        mControllers.add(ConnectivityController.get(this));
        mControllers.add(TimeController.get(this));
        mControllers.add(IdleController.get(this));
        mControllers.add(BatteryController.get(this));
        mControllers.add(AppIdleController.get(this));
        mControllers.add(ContentObserverController.get(this));
        mControllers.add(DeviceIdleJobsController.get(this));
    }

以ConnectivityController为例：
1.注册网络监听
    private ConnectivityController(StateChangedListener stateChangedListener, Context context,
            Object lock) {
        ...
        final IntentFilter intentFilter = new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION);
        mContext.registerReceiverAsUser(
                mConnectivityReceiver, UserHandle.SYSTEM, intentFilter, null, null);
        ...
    }
2.网络变化时，更新任务的状态，回调JobSchedulerService.onControllerStateChanged触发任务执行

    private BroadcastReceiver mConnectivityReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            updateTrackedJobs(-1);
        }
    };
    
    private void updateTrackedJobs(int uid) {
        synchronized (mLock) {
            boolean changed = false;
            for (int i = 0; i < mTrackedJobs.size(); i++) {
                final JobStatus js = mTrackedJobs.get(i);
                if (uid == -1 || uid == js.getSourceUid()) {
                    changed |= updateConstraintsSatisfied(js);
                }
            }
            if (changed) {
                mStateChangedListener.onControllerStateChanged();
            }
        }
    }

3.JobSchedulerService.onControllerStateChanged
    @Override
    public void onControllerStateChanged() {
        mHandler.obtainMessage(MSG_CHECK_JOB).sendToTarget();
    }

#四、总结  
通过两次跨进程的调用, 第一次是从app通过IJobScheduler调用到framework的JobScheduleService.schedule方法。 第二次则是framework 采用bindService方式通过IJobService接口，然后调用JobService的onStartJob，
该方法运行在app进程的主线程, 那么当存在耗时操作时则必须要采用异步方式, 让耗时操作交给子线程去执行,这样就不会阻塞app的UI线程。
JobStore 保存Job信息，设置为persist的Job会写入到配置文件/data/system/job/jobs.xml中，在开机启动时读取该文件，即使重启手机任务也可执行。
StateController 监听网络等的变化，触发任务的执行。

问题：
1.setPeriodic(REFRESH_INTERVAL) 、jobFinished（xxx，true）和 onStopJob（true）对任务重复的影响

参考资料：
http://wiki.jikexueyuan.com/project/android-weekly/issue-146/using-jobscheduler.html
http://www.bijishequ.com/detail/418422?p=
https://stackoverflow.com/questions/38344220/job-scheduler-not-running-on-android-n
http://gityuan.com/2017/03/10/job_scheduler_service/