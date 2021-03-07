# ThreadPoolExecutor 线程池原理

# git st一、为什么要使用线程池？

平时讨论多线程处理，大佬们必定会说使用线程池，那为什么要使用线程池？其实，这个问题可以反过来思考一下，不使用线程池会怎么样？

- 当需要多线程并发执行任务时，只能不断的通过new Thread创建线程，每创建一个线程都需要在堆上分配内存空间，同时需要分配虚拟机栈、本地方法栈、程序计数器等线程私有的内存空间。
- 当这个线程对象被可达性分析算法标记为不可用时被GC回收，这样频繁的创建和回收需要大量的额外开销。
- 再者说，JVM的内存资源是有限的，如果系统中大量的创建线程对象，JVM很可能直接抛出OutOfMemoryError异常，还有大量的线程去竞争CPU会产生其他的性能开销，更多的线程反而会降低性能，所以必须要限制线程数。

**使用线程池有哪些好处：**

- **降低资源消耗**：线程池可以复用池中的线程，不需要每次都创建新线程，减少创建和销毁线程的开销。
- **提高响应速度**：任务到达时，无需等待线程创建即可立即执行。
- **提高线程的可管理性**：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
- **线程池可实现线程环境的隔离**，例如分别定义支付功能相关线程池和优惠券功能相关线程池，当其中一个运行有问题时不会影响另一个。
- **提供更多更强大的功能**：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程池ScheduledThreadPoolExecutor，就允许任务延期执行或定期执行。

Java中的线程池核心实现类是ThreadPoolExecutor，下面来分析下它的基本原理。

# 二、7大核心参数

ThreadPoolExecutor构造方法中定义了7个参数，下面具体看下各个参数的含义。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

1. **coreSize**

   默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到任务队列当中。线程池将长期保证这些线程处于存活状态，即使线程已经处于闲置状态。除非配置了allowCoreThreadTimeOut=true，核心线程数的线程也将不再保证长期存活于线程池内，在空闲时间超过keepAliveTime后被销毁。

2. **maximumPoolSize**

   线程池内的最大线程数量，线程池内维护的线程不得超过该数量，大于核心线程数量小于最大线程数量的线程将在空闲时间超过keepAliveTime后被销毁。当阻塞队列存满后，将会创建新线程执行任务，线程的数量不会大于maximumPoolSize。maximumPoolSize的上限是

$$
2^n -1 (n = 29)
$$

3. **keepAliveTime** 

   存活时间，若线程数超过了corePoolSize，线程闲置时间超过了存活时间，该线程将被销毁。除非配置了allowCoreThreadTimeOut=true，核心线程数的线程也将不再保证长期存活于线程池内，在空闲时间超过keepAliveTime后被销毁。

4. **unit**

   线程存活时间的单位，例如TimeUnit.SECONDS表示秒。

5. **workQueue**

   线程池中保存等待执行的任务的阻塞队列。通过execute方法提交的Runable对象都会“存储”在该队列中，能够通过实现BlockingQueue接口来自定义我们所需要的阻塞队列。使用不同的队列可以实现不一样的任务存取策略，常见的阻塞队列有：

   ![image-20210304161843818](../../pics/image-20210304161843818.png)

6. **threadFactory**

   创建线程的工厂，默认是DefaultThreadFactory，可以定义线程名称，分组，优先级等。

7. **RejectedExecutionHandler**

   拒绝策略（饱和策略），当任务队列存满并且线程池个数达到maximunPoolSize后采取的策略。ThreadPoolExecutor中提供了四种拒绝策略如下：

   | 可选值              | 说明                                                   |
   | ------------------- | ------------------------------------------------------ |
   | AbortPolicy         | 直接抛出RejectedExecutionException异常，默认拒绝策略。 |
   | DiscardPolicy       | 不进行处理也不抛出异常。                               |
   | DiscardOldestPolicy | 丢弃队列里最前的任务，执行当前任务。                   |
   | CallerRunsPolicy    | 由调用线程执行该任务。                                 |

# 三、工作原理

ThreadPoolExecutor运行机制如下图所示：

![image-20210304172819822](../../pics/image-20210304172819822.png)

线程池在内部实际上构建了一个生产者消费者模型，将线程和任务两者解耦，并不直接关联，从而良好的缓冲任务，复用线程。线程池的运行主要分成两部分：任务管理、线程管理。

- 任务管理部分充当生产者的角色，当任务提交后，线程池会判断该任务后续的流转：（1）直接申请线程执行该任务；（2）缓冲到队列中等待线程执行；（3）拒绝该任务。

- 线程管理部分是消费者，它们被统一维护在线程池内，根据任务请求进行线程的分配，当线程执行完任务后则会继续获取新的任务去执行，最终当线程获取不到任务的时候，线程就会被回收。

我们会按照以下三个部分去详细讲解线程池运行机制：

1. 线程池如何维护自身状态。
2. 线程池如何管理任务。
3. 线程池如何管理线程。

## 3.1、线程池状态管理

线程池内部使用一个变量**ctl**维护运行状态(runState)和线程数量 (workerCount)，是一个32位二进制数，其中前3位表示线程池状态，后29位表示线程数。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

线程池状态：

| 状态       | 前3位值 | 描述                                                         |
| ---------- | ------- | ------------------------------------------------------------ |
| RUNNING    | 111     | 能接受新提交的任务，也能处理阻塞队列中的任务。               |
| SHUTDOWN   | 000     | 关闭状态，不再接受新提交的任务，单可以继续执行已添加到阻塞队列中的任务。 |
| STOP       | 001     | 不能接受新任务，也不能处理队列中的任务，会中断正在执行的任务的线程。 |
| TIDYING    | 010     | 所有的任务都已经终止，workerCount（有效线程数）为0           |
| TERMINATED | 011     | 在terminated()方法执行完后进入该状态                         |

生命周期的转换如下图：

![image-20210304172419652](../../pics/image-20210304172419652.png)



线程池运行的状态，并不是用户显式设置的，而是伴随着线程池的运行，主要由内部来维护。

## 3.2、任务执行

向线程池中提交一个任务后，线程池的处理流程如下：

![image-20210304164400691](../../pics/image-20210304164400691.png)

1. 如果线程池中的数量未达到核心线程的数量，则直接会启动一个核心线程来执行任务。
2. 如果线程池中的数量已经达到或超过核心线程的数量，则任务会被插入到任务队列中等待执行。

3. 如果任务队列已满，且线程数未达到最大线程数，则会启动一个非核心线程来执行任务。

4. 如果线程数已达到最大值，按照饱和策略执行该任务。

具体代码实现execute方法中：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        // 1、如果线程池中的数量未达到核心线程的数量，则直接会启动一个核心线程来执行任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 2、如果线程池中的数量已经达到或超过核心线程的数量，则任务会被插入到任务队列中等待执行
    if (isRunning(c) && workQueue.offer(command)) {
        // 再次检查线程池的状态
        int recheck = ctl.get();
        // 2.1、线程池已经终止运行且任务可以从线程池中移除掉，执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 2.2、线程池中无线程，创建线程执行
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 3、如果任务队列已满，且线程数未达到最大线程数，则会启动一个非核心线程来执行任务。
    else if (!addWorker(command, false))
        // 4、如果线程数已达到最大值，按照饱和策略执行该任务。
        reject(command);
}
```

### 3.2.1、addWorker

增加线程并执行，最后返回是否成功。addWorker方法有两个参数：firstTask、core。firstTask参数用于指定新增的线程执行的第一个任务，该参数可以为空；core参数用于判断新增线程时的阈值是corePoolSize还是maximumPoolSize，代码如下：

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 1、检查线程池是否不可创建任务
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                // 2、检查线程数是否超过阈值
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 3、增加线程数是否成功
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                // 4、重新检查线程池状态，防止添加任务过程中，线程池状态被SHUTDONW
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 5、检查线程状态
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    // 6、检查线程状态，线程池在running或是正在退出时，若线程已经运行中则抛出异常。
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 7、添加worker到hashSet中
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 8、启动工作线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

其执行流程如下图所示：

![image-20210307210048523](../../pics/image-20210307210048523.png)

经过上面的分析，我们知道线程池将线程和任务封装到Worker中，下面看下worker的实现。

### 3.2.2、worker

Worker封装了thread和firstTask，thread是在调用构造方法时通过ThreadFactory来创建的线程，可以用来执行任务；firstTask用它来保存传入的第一个任务，这个任务可以有也可以为null。如果firstTask != null，那么线程就会在启动初期立即执行这个任务，对应是核心线程创建；如果first == null，则创建一个线程去执行任务列表（workQueue）中的任务，也就是非核心线程的创建。

Worker执行任务的模型如下图所示：

![image-20210307221006057](../../pics/image-20210307221006057.png)

```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // worker 实现了Runnable接口，thread.start的时候执行run方法
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker. */
    public void run() {
        runWorker(this);
    }
		// ... 
}
```

worker 实现了Runnable接口，thread.start的时候执行run方法，再调用到runWorker。

### 3.2.3、runWorker

runWorker方法中，使用循环，通过getTask方法，不断从阻塞队列中获取任务执行，如果任务不为空则执行任务，这里实现了**线程的复用，不断的获取任务执行，不用重新创建线程**。

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 1.若firstTask!=null，执行firstTask；firstTask!=null,从队列中取第一个任务执行
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 2.线程池若处于STOP状态时，则中断当前线程。 
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 3.如果getTask结果为null则跳出循环，执行processWorkerExit()方法，销毁线程。
        processWorkerExit(w, completedAbruptly);
    }
}
```

43行：如果getTask结果为null则跳出循环，执行processWorkerExit()方法，销毁线程·1。那getTask何时返回null？接着看getTask源码。

### 3.2.4、getTask

getTask主要的职责是不断从workQueue中取任务出来，同时会判断当前线程池的状态和线程数。

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 1.（线程状态 == STOP或TIDYING或TERMINATED） || （线程状态为SHUTDOWN && 任务队列为空）返回空，可以关闭线程池
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // 2.是否要回收线程
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // 3.线程数 > max 或 队列为空
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 4.获取队列中的任务，若回收线程，则等待keepAliveTime时间再从队列中获取
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

getTask的流程图如下：

![image-20210307225155723](../../pics/image-20210307225155723.png)

### 3.2.5、processWorkerExit

前面我们说到，当任务获取不到时会从尝试回收线程，具体代码如下：

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        // 1.移除worker
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
		// 2、终止当前线程
    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

## 3.3、任务结束

# 四、Executors

除了自定义线程池外，JDK的Executors类中还定义了几种线程池如下：

| 线程                    | 描述                                                         | 阻塞队列            |
| ----------------------- | ------------------------------------------------------------ | ------------------- |
| newFixedThreadPool      | 只有核心线程，核心线程数 == 最大线程数，                     | LinkedBlockingQueue |
| newCachedThreadPool     | 只有非核心线程，最大线程数为Integer.MAX_VALUE                | SynchronousQueu     |
| newSingleThreadExecutor | 线程池内部只有一个核心线程，所有的任务都在一个线程中执行，用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行 | LinkedBlockingQueue |
| newScheduledThreadPool  | 最大线程数为Integer.MAX_VALUE，执行定时任务和固定周期的任务  | DelayedWorkQueue    |
| newScheduledThreadPool  | 和上面区别是该线程池大小为1。                                | DelayedWorkQueue    |

一般情况下不推荐使用Executors创建，建议通过ThreadPoolExecutor的方式创建，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。Executors返回的线程池对象的弊端如下：

1. FixedThreadPool和SingleThreadPool：允许的请求队列长度为Integet.MAX_VALUE,可能会堆积大量的请求从而导致OOM;

2. CachedThreadPool：允许创建线程数量为Integet.MAX_VALUE,可能会创建大量的线程，从而导致OOM.

参考：
http://blog.csdn.net/u010687392/article/details/49850803
《Android开发艺术探索》

https://www.cnblogs.com/franson-2016/p/13291591.html

https://www.cnblogs.com/zhangziqiu/archive/2011/03/30/ComputerCode.html