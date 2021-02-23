

## 一、从 Handler.sendMessage 说起：

Android UI是线程不安全的，如果在子线程中尝试进行UI操作，程序就有可能会崩溃。要处理这样的问题就需要通过handler的异步消息处理机制了，通常有2种写法：
1.sendMessage

    public class MainActivity extends AppCompatActivity {
        private Handler mHandler;
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            mHandler = new Handler(){
                @Override
                public void handleMessage(Message msg) {
                    //收到消息后，更新 ui
                    }
            };
            
            new Thread(new Runnable() {
                    @Override
                    public void run() {
                        //发送消息
                        mHandler.sendMessage();
                    }
                }).start();
            }
    }

2.post
    
    ...
    mHandler = new Handler();
    new Thread(new Runnable() {
           @Override
           public void run() {
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        //ui操作
                    }
                }); 
           }
       }).start();
    ...

其实这2种方式最终都是殊途同归：最终调用的都是Handler的sendMessageAtTime方法
    
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

不同地方在于post 将Runnable封装到Message中，赋值给msg.callback：

    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

最后在dispatchMessage处理消息时，sendMessage会执行handleMessage，而post方式msg.callback不为空，看上面代码的第8行，会执行handleCalback，最终回调Runnable的run方法。

    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCalback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
    
    private static void handleCallback(Message message) {
        message.callback.run();
    }
细心的读者可能会问mCallback又是什么呢？其实mCallback就是你在new Handler时可以传的一个参数，如果你像下面这样在Handler.Callback中返回true，就可以拦截消息不让消息在Handler的handler的handleMessage中处理
    
    mHandler = new Handler(new Handler.Callback() {
                @Override
                public boolean handleMessage(Message msg) {
                    //可以拦截
                    return true;
                }
            }){
                @Override
                public void handleMessage(Message msg) {
                
                }
            };

## 二、sendMessage之后发生了什么？
回到sendMessageAtTime方法中，可以看到会调用MessagQueue.enqueueMessage，MessageQueue就是我们常说的消息队列， 用于将所有收到的消息以队列的形式进行排列，并提供入队和出队的方法。通过enqueueMessage就将我们发的消息加入到了消息队列中。

    boolean enqueueMessage(Message msg, long when) {
        ...
        synchronized (this) {
            ...
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
        ...
    }

通过以上我们知道了入队，那又该如何出队呢？这就要涉及到Looper了，我们在new 一个Handler的时候，Handler的构造方法中就会初始化一个Looper

    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        mLooper = Loope.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    
    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

代码第10行，我们可以看到handler 中的mlooper是通过Loope.myLooper()获取，同时我们要注意到Handler的mQueue其实就是Looper的mQueue！继续跟踪到Looper.myLooper，认真看注释，如果sThreadLocal中有Looper存在就返回Looper，如果没有Looper存在自然就返回空了。因此你可以想象得到是在哪里给sThreadLocal设置Looper了吧，那就是Looper.prepare()方法Looper.prepare()在哪里被调用的，我们先放到一边，继续回到我们的上个话题，消息通过Looper怎么出队?请看Looper.loop()。

三、Looper.loop()
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
    
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            
            ...
            
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            ...
        }
       msg.recycleUnchecked();
    }

可以看到，这个方法从第6行开始，进入了一个死循环，然后不断地调用的MessageQueue的next()方法，这个next()方法就是消息队列的出队方法。它的简单逻辑就是如果当前MessageQueue中存在mMessages(即待处理消息)，就将这个消息出队，然后让下一条消息成为mMessages，否则就进入一个阻塞状态，一直等到有新的消息入队。继续看loop()方法的第14行，每当有一个消息出队，就将它传递到msg.target的dispatchMessage()方法中，那这里msg.target又是什么呢？其实就是Handler啦，你观察一下上面sendMessageAtTime()方法时调用的enqueueMessage方法的第2行就可以看出来了，出队完成！

第24行消息回收过程，这里涉及到Message的回收和复用，后面再讲。

总结一下：loop方法其实就是建立一个死循环，然后从消息队列中逐个取出消息，进行处理和回收的过程。

## 四、Looper.prepare到底做了什么事？

    public static void prepare() {
        prepare(true);
    }
    
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

第5行 - 第15行可以看出，调用Looper.prepare过程中会将当前线程和Looper绑定在一起，所以不能直接在子线程中直接new Handler 会抛出异常  "Can't create handler inside thread that has not called Looper.prepare()"
    
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

也就是说，new Hanlder 之前必须要调用 Looper.prepare方法，那为何在主线程中使用是不需要呢？
其实Android在程序启动的时候，系统已经帮我们自动调用了Looper.prepare()方法。查看ActivityThread中的main()方法，代码如下所示：

    public static void main(String[] args) {
        ...
        Looper.prepareMainLooper();
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }
        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();
        ...
    }

可以看到上面代码第3行调用了 Looper.prepareMainLooper()，再调用到Looper的prepare，最终将主线程和Lopper、Thread以及MessageQueue绑定在一起。    
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
    
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

一个最标准的异步消息处理线程的写法应该是这样

    class LooperThread extends Thread {  
          public Handler mHandler;  
      
          public void run() {  
              Looper.prepare();  
      
              mHandler = new Handler() {  
                  public void handleMessage(Message msg) {  
                      // process incoming messages here  
                  }  
              };  
      
              Looper.loop();  
          }  
      }    

这样基本就将Handler的创建过程完全搞明白了，总结一下就是在主线程中可以直接创建Handler对象，而在子线程中需要先调用Looper.prepare()才能创建Handler对象。

## 五、Message 和 享元模式

通常我们获取一个Message都是通过Message的obtain方法：

   public final class Message implements Parcelable {
    ...
    // sometimes we store linked lists of these things
    /*package*/ Message next;

    private static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;
    private static final int MAX_POOL_SIZE = 50;
    
    /**
      * Return a new Message instance from the global pool. Allows us to
      * avoid allocating new objects in many cases.
      */
     public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    ...
  }
每一个Message对象都有一个同类型的next字段，这个next字段指向的就是下一个可用的Message，最后一个可用的Message字段的next为空，这样所有的Message对象就通过next串联成一个Message池了，而sPool 永远指向的是整个链表的第一个元素。
13 - 18行，当使用obtain方法获取一个Message对象时，其实获取的就是链表中的第一个元素，同时将sPool在指向下一个Message。
那么这些Message对象是在什么时候被放到链表中的呢，在Message类的说明中有这样一句话： While the constructor of Message is public, the best way to getone of these is to call {@link #obtain Message.obtain()} or one of the methods, which will pull them from a pool of recycled objects。原来在创建Message时不会将Message放入队列而是在recycle时才会加入到队列，让我们先来看下recylce方法：

     /**
     * Return a Message instance to the global pool.
     * <p>
     * You MUST NOT touch the Message after calling this function because it has
     * effectively been freed.  It is an error to recycle a message that is currently
     * enqueued or that is in the process of being delivered to a Handler.
     * </p>
     */
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }
    
    /**
     * Recycles a Message that may be in-use.
     * Used internally by the MessageQueue and Looper when disposing of queued Messages.
     */
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;
    
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }

10 - 17行，判断Message是否在被使用中，如果没有，则执行回收操作。
27 - 43行，先清空Message的各字段，并将flag置为FLAG_IN_USE，这个flag在obtain时会被置为0。然后继续判断是否要将该消息放到回收池中，如果池的大小小于MAX_POOL_SIZE（50），那么就进行链表操作，将这个Message放到链表的表头。

总结一下：Message 通过在内部构建一个链表维护一个被回收的Message对象的对象池，当用户调用obtain函数时优先从池中获取，如果池中没有可以复用的对象则创建一个新的Message对象。这些新创建的Message对象再被使用完之后会被回收到这个对象池中，当下次再调用obtain函数时，他们就会被复用。
Message对象没有内部外部的状态区别，更像是一个对象池。结合的Looper.loop方法，在使用完一个Message对象后就将会将它回收，避免系统中创建太多Message对象。

## 六、子线程中更新UI的
除了Handler的sendMessage和post之外，我们还有以下2种方法可以在子线程中进行UI操作，一句话解释完，请看注释！！！
    
1. View的post()方法

        /**
         * <p>Causes the Runnable to be added to the message queue.
         * The runnable will be run on the user interface thread.</p>
         */
        public boolean post(Runnable action) {
            final AttachInfo attachInfo = mAttachInfo;
            if (attachInfo != null) {
                return attachInfo.mHandler.post(action);
            }
        
            // Postpone the runnable until we know on which thread it needs to run.
            // Assume that the runnable will be successfully placed after attach.
            getRunQueue().post(action);
            return true;
        }

2. Activity的runOnUiThread()方法

         /**
         * Runs the specified action on the UI thread. If the current thread is the UI
         * thread, then the action is executed immediately. If the current thread is
         * not the UI thread, the action is posted to the event queue of the UI thread.
         *
         * @param action the action to run on the UI thread
         */
        public final void runOnUiThread(Runnable action) {
            if (Thread.currentThread() != mUiThread) {
                mHandler.post(action);
            } else {
                action.run();
            }
        }


## 总结：
Looper和thread以及messageQueue都是一一对应的，而一个Handler只能关联一个Looper，一个Looper可以关联多个Handler, handler的messagequeue就是Looper的messagequeue。  

这篇文章主要参考了郭霖大神的博客，按照自己的思路和理解写的，以下是自己还未搞懂的问题，欢迎讨论！

问题：
1.mCallback的用法？
2.Handler(Callback callback, boolean async)  async 是什么意思，真正的异步，不添加队列？async 如果设置为true，那么不会通过队列发送消息？
3.post 方式run里面不能做耗时任务？
    会产生anr，post的Runnable是在主线程执行的。
4.view.post 为何会在onCreate中执行时为何会无效？
5.ThreadgetLocal.set 和 get的逻辑？
6.不同的线程是不是使用相同的Message对象？
7.navtive层的原理。
参考资料：
http://blog.csdn.net/guolin_blog/article/details/9991569