## 一、用法
### 1.类定义：
三种泛型类型分别代表“启动任务执行的输入参数”、“后台任务执行的进度”、“后台计算结果的类型”。在特定场合下，并不是所有类型都被使用，如果没有被使用，可以用Java.lang.Void类型代替。
    
    public abstract class AsyncTask<Params, Progress, Result> {...}

### 2.方法说明    

除了doInBackground是在线程池中执行，其他方法均在主线程执行。
    
1.execute(Params... params)，执行一个异步任务，需要我们在代码中调用此方法，触发异步任务的执行。
    
2.onPreExecute()，在execute(Params... params)被调用后立即执行，一般用来在执行后台任务前对UI做一些标记。
    
3.doInBackground(Params... params)，在onPreExecute()完成后立即执行，用于执行较为费时的操作，此方法将接收输入参数和返回计算结果。在执行过程中可以调用publishProgress(Progress... values)来更新进度信息。
    
4.onProgressUpdate(Progress... values)，在调用publishProgress(Progress... values)时，此方法被执行，直接将进度信息更新到UI组件上。
    
5.onPostExecute(Result result)，当后台操作结束时，此方法将会被调用，计算结果将做为参数传递到此方法中，直接将结果显示到UI组件上。
    
6.onCancelled() 异步任务被取消时执行。
    
### 3.注意
1.异步任务的实例必须在UI线程中创建。

2.execute(Params... params)方法必须在UI线程中调用。

3.不要手动调用onPreExecute()，doInBackground(Params... params)，onProgressUpdate(Progress... values)，onPostExecute(Result result)这几个方法。

4.不能在doInBackground(Params... params)中更改UI组件的信息，一般都是通过publishProgress(i)，在onProgressUpdate中操作UI线程的变化，或者doInBackground结束后，在onPostExecute中返回结果

5.一个任务实例只能执行一次，如果执行第二次将会抛出异常。
    
6.多个任务默认是线性执行的，保证提交的任务确实是按照先后顺序执行的。它的内部有一个队列用来保存所提交的任务，保证当前只运行一个，这样就可以保证任务是完全按照顺序执行的，默认的execute()使用的就是这个，也就是executeOnExecutor(AsyncTask.SERIAL_EXECUTOR)与execute()是一样的。
默认还提供了一个并发执行的线程池
THREAD_POOL_EXECUTOR是一个有限制数目的线程池 （8 核 9个，看具体逻辑）的线程池，也就是说最多只有5个线程同时运行，超过的就要等待。
    
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    // We want at least 2 threads and at most 4 threads in the core pool,
    // preferring to have 1 less than the CPU count to avoid saturating
    // the CPU with background work
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR;
    
    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
如果你想让所有的任务都能并发同时运行，那就创建一个没有限制的线程池(Executors.newCachedThreadPool())，并提供给AsyncTask。这样这个AsyncTask实例就有了自己的线程池而不必使用AsyncTask默认的。

    new Task().executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
## 二、源码分析

    重要变量：
    
    private static volatile Executor sDefaultExecutor 用于任务排队的线程池
    
    public static final Executor THREAD_POOL_EXECUTOR 具体执行任务的线程池
    
    private static InternalHandler sHandler 从线程池切换到主线程
    
    private final WorkerRunnable<Params, Result> mWorker 实现了Callable接口，这里的Param和Result是AsyncTask定义的泛型类型
    
    private final FutureTask<Result> mFuture 
    FutureTask类其实是实现了Future和Runnable接口，具备了这两个接口的功能


### 1.从new AsyncTask().execute()开始说起
执行Async通常有2个方法供我们开启异步任务，分别是：
    
    execute(Params... params)
    executeOnExecutor(Executor exec, Params... params)
这个2个方法的区别就在于是否使用默认的线程池执行，使用者可根据实际的业务需求定义合适的线程池

    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }
    
        mStatus = Status.RUNNING;
    
        onPreExecute();
    
        mWorker.mParams = params;
        exec.execute(mFuture);
    
        return this;
    }

在代码第9-21行：判断当前的任务是不是在队列里，如果不是再去判断是不是正在执行或者已经执行完了，保证同一个任务实例只会执行一次，如果是第一次执行，那么就将它的状态置为running。
22行 执行 onPreExecute 中的操作
23行 将传入的参数Params赋值给mWorker的mParams，真正的执行会调用到mWorker
26行 通过Executor执行mFuture，在代码的第3行可以看到默认使用的Executor是sDefaultExecutor

### 2. sDefaultExecutor.execute？

    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;
    
        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }
    
        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

12行，exce方法会将传入的mFuture的run方法在封装到一个Runnable对象中，并且将这个对象通过mTasks.offer加入到类型为ArrayDeque的任务队列mTask中，

     /**
     * Inserts the specified element at the end of this deque.
     *
    public boolean offer(E e) {
        return offerLast(e);
    }

22行，第一次执行时，mActive一定为空，那么scheduleNext一定会被执行，最终会执行29行THREAD_POOL_EXECUTOR.execute(mActive)，在线程池中执行该任务

总结下，到这里我们已经知道2个重要的点：
    1.传入的参数是被封装到mWorker里
    2.THREAD_POOL_EXECUTOR 执行的其实是mFuture的run方法

### 3.mFuture 和 mWorker？

mFuture是个什么呢？
    
    private final FutureTask<Result> mFuture;

这是一个实现了Runnable接的类,那么我们先看下它的run方法吧

    public void run() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

代码第6-12行，我们可以看到执行了一个callable.call()方法，并且在第19行将返回值set(result);
这个callable个什么呢？

    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

原来是构造函数传递进来的值，那么是在什么时候别初始化的呢？请看AsyncTask()的构造函数。

    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };
        
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

第24行， 哦！原来在我们new AsyncTask的时候我们就初始化了一个mWorker对象，并且将该参数传递给了mFuture，callable原来就是mWorker，
第12行， result = doInBackground(mParams); mWorker.call方法原来会调用到doInBackground，原来我们所有传递过来的参数都是通过mWorker传递到doInBackground中的
第18行， 将返回值通过post到主线程，这里就用到了InternalHandler

##4. InternalHandler线程池到主线程

    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
    
    private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }
    
    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }
    
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

好了，到这里一个AsyncTask的执行就介绍完了，下面介绍下任务的进度更新和取消

### 5. publishProgress 
我们知道可以通过在doinBackground中调用publishProgress来实时更新UI，那么是如何实现的呢？

    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }

原来它也是通过InternalThread发送消息，结合InternalHandler中的代码，可知最终通过onProgressUpdate方法实现ui更新

### 6.关于任务取消
boolean cancel(boolean mayInterruptIfRunning)：试图取消对此任务的执行。如果任务已完成、或已取消，或者由于某些其
他原因而无法取消，则此尝试将失败。当调用 cancel 时，如果调用成功，而此任务尚未启动，则此任务将永不运行。如果任务已经
启动，则 mayInterruptIfRunning 参数确定是否应该以试图停止任务的方式来中断执行此任务的线程。此方法返回后，对 isDone() 的后续调用将始终返回 true。如果此方法返回 true，则对 isCancelled() 的后续调用将始终返回 true。 


问题：
1.正常执行过程中，在mFuture中的set(result)方法吗？到底做了什么事情呢？感觉像是将task和对应的future标记为已经执行了的。

2.mfuture 内部涉及的原理？

参考：
http://blog.csdn.net/liuhe688/article/category/790276
http://blog.csdn.net/jiangwei0910410003/article/details/40113055


​    