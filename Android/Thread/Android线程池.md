## 一、为什么要使用线程池？
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

## 二、ThreadPoolExecutor

1、7大核心参数

2、



既然线程池就是ThreadPoolExecutor，所以我们要创建一个线程池只需要new ThreadPoolExecutor(…);就可以创建一个线程池，而如果这样创建线程池的话，我们需要配置一堆东西，非常麻烦，我们可以看一下它的构造方法就知道了：
    

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }

所以，官方也不推荐使用这种方法来创建线程池，而是推荐使用Executors的工厂方法来创建线程池，Executors类是官方提供的一个工厂类，它里面封装好了众多功能不一样的线程池，从而使得我们创建线程池非常的简便，主要提供了如下五种功能不一样的线程池：
    

.ThreadPoolExecutor解析

我们看到通过Executors的工厂方法来创建线程池极其简便，其实它的内部还是通过new ThreadPoolExecutor(…)的方式创建线程池的，所以我们要了解线程池还是得了解ThreadPoolExecutor这个线程池类，其中由于和定时任务相关的线程池比较特殊（newScheduledThreadPool()、newSingleThreadScheduledExecutor()），它们创建的线程池内部实现是由ScheduledThreadPoolExecutor这个类实现的，而ScheduledThreadPoolExecutor是继承于ThreadPoolExecutor扩展而成的，所以本质还是一样的，只不过多封装了一些定时任务相关的api，所以我们主要就是要了解ThreadPoolExecutor，从构造方法开始：

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
    
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
参数说明： 
### 1.简单参数 
corePoolSize：线程池中的核心线程数量，核心线程会在线程池中一直存在，如果设置了allowCoreThreadTimeOut(true)，超时策略也会生效，即核心线程在等待超过keepAliveTime后也会被终止。 maximumPoolSize：线程池中的最大线程数量，当活动的线程超过了这个数量时，后续的任务会被阻塞。 keepAliveTime：就是当线程池中的线程数量超过了corePoolSize时，非核心线程在超过keepAliveTime时间内没有任务的话则被销毁。这个主要应用在缓存线程池中 
unit：它是一个枚举类型，表示keepAliveTime的单位，常用的如：TimeUnit.SECONDS（秒）、TimeUnit.MILLISECONDS（毫秒）
threadFactory：线程工厂，用来创建线程池中的线程，通常用默认的即可
### 2.BlockingQueue<Runnable> workQueue
workQueue：任务队列，主要用来存储已经提交但未被执行的任务，不同的线程池采用的排队策略不一样，java中提供的workQueue：

    1.ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。
    2.LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素，吞吐量通常要高于ArrayBlockingQueue(容量Integer.MaxValue)
    3.SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，
    4.PriorityBlockingQueue：一个具有优先级的无限阻塞队列。
    5.DelayedWorkQueue：等待队列，因为这些任务是带有延迟的，而每次执行都是取第一个任务执行，因此在DelayedWorkQueue中任务必然按照延迟时间从短到长来进行排序的。

### 3.五种线程池分别用的lockingQueue：

    1、newFixedThreadPool()—>LinkedBlockingQueue 
    2、newSingleThreadExecutor()—>LinkedBlockingQueue 
    3、newCachedThreadPool()—>SynchronousQueue 
    4、newScheduledThreadPool()—>DelayedWorkQueue 
    5、newSingleThreadScheduledExecutor()—>DelayedWorkQueue 等待队列

### 4.RejectedExecutionHandler
通常叫做拒绝策略，当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。以下是JDK1.5提供的四种策略。

    1.AbortPolicy：直接抛出异常。
    2.CallerRunsPolicy：只用调用者所在线程来运行任务。
    3.DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
    4.DiscardPolicy：不处理，丢弃掉。
当然也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化不能处理的任务。

### 2. 运行规则

（1）如果线程池中的数量为达到核心线程的数量，则直接会启动一个核心线程来执行任务。

（2）如果线程池中的数量已经达到或超过核心线程的数量，则任务会被插入到任务队列中标等待执行。

（3）如果(2)中的任务无法插入到任务队列中，由于任务队列已满，这时候如果线程数量未达到线程池规定最大值，则会启动一个非核心线程来执行任务。

（4）如果(3)中线程数量已经达到线程池最大值，则会拒绝执行此任务，ThreadPoolExecutor会调用RejectedExecutionHandler的rejectedExecution方法通知调用者

单次可执行任务数：maximumPoolSize



## 三、ExecutorService 

通过上述分析，我们知道了通过new Thread().start()方式创建线程去处理任务的弊端，而为了解决这些问题，Java为我们提供了ExecutorService线程池来优化和管理线程的使用。ExecutorService，是一个接口，其实如果要从真正意义上来说，它可以叫做线程池的服务，因为它提供了众多接口api来控制线程池中的线程，而真正意义上的线程池就是：ThreadPoolExecutor，它实现了ExecutorService接口，并封装了一系列的api使得它具有线程池的特性，其中包括工作队列、核心线程数、最大线程数等。
**使用线程池的优点**
1、线程的创建和销毁由线程池维护，一个线程在完成任务后并不会立即销毁，而是由后续的任务复用这个线程，从而减少线程的创建和销毁，节约系统的开销
2、线程池旨在线程的复用，这就可以节约我们用以往的方式创建线程和销毁所消耗的时间，减少线程频繁调度的开销，从而节约系统资源，提高系统吞吐量
3、在执行大量异步任务时提高了性能

### 1、newFixedThreadPool

该模式全部由核心线程去实现，并不会被回收，没有超时限制和任务队列的限制，会创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
        

```java
ExecutorService fixedThreadPool = Executors.newFixedThreadPool(5);
```

### 2、newCachedThreadPool

该模式下线程数量不定的线程池，只有非核心线程，最大值为Integer.MAX_VALUE，会创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。

```java
  ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
```

### 3、newSingleThreadExecutor

该模式下线程池内部只有一个核心线程，所有的任务都在一个线程中执行，会创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。实现代码如下：

```java
ExecutorService singleThreadPool = Executors.newSingleThreadExecutor();
```

### 4、newScheduledThreadPool

该模式下核心线程是固定的，非核心线程没有限制，非核心线程闲置时会被回收。会创建一个定长线程池，执行定时任务和固定周期的任务。

```java
  ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
```

### 5、newSingleThreadScheduledExecutor()

该模式下返回一个可以控制线程池内线程定时或周期性执行某任务的线程池。只不过和上面的区别是该线程池大小为1，而上面的可以指定线程池的大小
    

```java
ScheduledExecutorService singleThreadScheduledPool = Executors.newSingleThreadScheduledExecutor();
```

#### ScheduledExecutorService可调用方法的说明

```java
1.schedule(Runnable command, long delay, TimeUnit unit)
延迟delay时间后执行，只执行一次。
         
2.scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnitunit)
if(线程执行时间＞period）
    执行周期 = 线程执行时间。
 else
    执行周期 = period（延迟）。
    
3.scheduleWithFixedDelay(Runnable command, long initialDelay, long delay,TimeUnit unit)
执行周期 = 线程执行时间+delay
```

## 



## 5.线程池其他常用功能

1.shutDown()  关闭线程池，不影响已经提交的任务
2.shutDownNow() 关闭线程池，并尝试去终止正在执行的线程
3.allowCoreThreadTimeOut(boolean value) 允许核心线程闲置超时时被回收
4.submit() 一般情况下我们使用execute来提交任务，但是有时候可能也会用到submit，使用submit的好处是submit有返回值

    public void submit(View view) {
        List<Future<String>> futures = new ArrayList<>();
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(3, 5, 1,
                TimeUnit.SECONDS, new LinkedBlockingDeque<Runnable>());
        for (int i = 0; i < 10; i++) {
            Future<String> taskFuture = threadPoolExecutor.submit(new MyTask(i));
            //将每一个任务的执行结果保存起来
            futures.add(taskFuture);
        }
        try {
            //遍历所有任务的执行结果
            for (Future<String> future : futures) {
                Log.d("google_lenve_fb", "submit: " + future.get());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
    
    class MyTask implements Callable<String> {
        private int taskId;
        
        public MyTask(int taskId) {
            this.taskId = taskId;
        }
        @Override
        public String call() throws Exception {
            SystemClock.sleep(1000);
            //返回每一个任务的执行结果
            return "call()方法被调用----" + Thread.currentThread().getName() + "-------" + taskId;
        }
    }

参考资料：
http://blog.csdn.net/u010687392/article/details/49850803
《Android开发艺术探索》

https://www.cnblogs.com/franson-2016/p/13291591.html

https://www.cnblogs.com/zhangziqiu/archive/2011/03/30/ComputerCode.html