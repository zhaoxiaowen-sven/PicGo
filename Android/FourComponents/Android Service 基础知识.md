# Android Service 基础知识

Service 作为android 四大组件之一，主要用于再后台处理一些耗时的逻辑或者去执行一些长期运行的任务。

# 一、startService 和 bindService

## 1. 生命周期和启动方式：
![service_lifecircle](https://leanote.com/api/file/getImage?fileId=595288b4ab6441560e001f51)
1.1 启动
通过startService启动服务时，如果是第一次启动，会调用onCreate->onStartCommand，但是多次启动时不会再调用onCreate，只有onStartCommand会被调用多次。
通过bindService绑定服务时，如果服务还未启动，会调用onCreate->onBind，创建并且绑定服务，多次调用bindService并不会多次调用onBind()，除非是多个客户端来绑定服务。

1.2 销毁
如果组件通过调用 startService() 启动服务，则服务将一直运行，直到服务使用 stopSelf()自行停止运行，或由其他组件通过调用 stopService() 停止它为止。无论服务被启动了多少次，只要调用一次stopService，便可终止（不考虑被绑定过的情况）。

如果组件是通过调用 bindService()来创建服务（且未调用onStartCommand()），则服务只会在该组件与其绑定时运行。一旦该服务与所有客户端之间的绑定全部取消，系统便会销毁它。

## 2. starService和bindService一起使用：
如果一个Service又被启动又被绑定，则该Service会一直在后台运行。首先不管如何调用，onCreate始终只会调用一次。startService调用多少次，Service的onStartCommand方法便会调用多少次。Service的终止，需要unbindService和stopService同时调用才行。不管startService与bindService的调用顺序，如果先调用unbindService，此时服务不会自动终止，再调用stopService之后，服务才会终止；如果先调用stopService，此时服务也不会终止，再次调用unbindService或者**之前调用bindService的Context不存在了**（如Activity被finish的时候）之后，服务才会停止。

## 3. 使用场景：
   startService 主要用于开启服务，例如后台下载和播放音乐
   bindService 主要为了使用服务中的一些功能或者与服务进行交互
   这2种模式不是完全分离的。你可以可以绑定到一个通过startService()启动的服务。如一个intent想要播放音乐，通过startService启动后台播放音乐的service。然后，也许用户想要操作播放器或者获取当前正在播放的乐曲的信息，一个activity就会通过bindService建立一个到此service的连接. 这种情况下 stopService() 在全部的连接关闭后才会真正停止service。

### Tips：
1. 服务的onCreate和onDestroy在一个生命周期中只会执行一次
2. service的stopself方法的功能是，当完成所有功能之后，将service停掉
3. Service中onRebind方法被调用的时机，需要满足2个条件：
   (1)服务中onUnBind方法返回值为true
   (2)服务对象被解绑后没有被销毁，之后再次被绑定

## 2. onStartCommand的返回值：

请注意，onStartCommand() 方法必须返回整型数。整型数是一个值，用于描述系统应该如何在服务终止的情况下继续运行服务（如上所述，IntentService 的默认实现将为您处理这种情况，不过您可以对其进行修改）。从onStartCommand()返回的值必须是以下常量之一：

## 1. START_NOT_STICKY
如果系统在 onStartCommand() 返回后终止服务，则除非有挂起 Intent 要传递，否则系统不会重建服务。这是最安全的选项，可以避免在不必要时以及应用能够轻松重启所有未完成的作业时运行服务。

## 2. START_STICKY
如果系统在 onStartCommand() 返回后终止服务，则会重建服务并调用 onStartCommand()，但不会重新传递最后一个 Intent。相反，除非有挂起 Intent 要启动服务（在这种情况下，将传递这些 Intent ），否则系统会通过空 Intent 调用 onStartCommand()。这适用于不执行命令、但无限期运行并等待作业的媒体播放器（或类似服务）。

## 3. START_REDELIVER_INTENT
如果系统在 onStartCommand() 返回后终止服务，则会重建服务，并通过传递给服务的最后一个 Intent 调用 onStartCommand()。任何挂起 Intent 均依次传递。这适用于主动执行应该立即恢复的作业（例如下载文件）的服务。

# 二、前台服务
Service几乎都是在后台运行的，一直以来它都是默默地做着辛苦的工作。但是Service的系统优先级还是比较低的，当系统出现内存不足情况时，就有可能会回收掉正在后台运行的Service。如果你希望Service可以一直保持运行状态，而不会由于系统内存不足的原因导致被回收，就可以考虑使用前台Service。前台Service和普通Service最大的区别就在于，它会一直有一个正在运行的图标在系统的状态栏显示，下拉状态栏后可以看到更加详细的信息，非常类似于通知的效果。当然有时候你也可能不仅仅是为了防止Service被回收才使用前台Service，有些项目由于特殊的需求会要求必须使用前台Service，比如说墨迹天气。

        Intent intent = new Intent(this, MainActivity.class);
        //需要让Activity运行在新的任务栈中
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        Notification.Builder builder = new Notification.Builder(this);
        Notification notification = builder.setSmallIcon(R.mipmap.ic_launcher)
                .setContentText("this is a notify")
                .setContentTitle("notify!!!").setTicker("notify")
                .setContentIntent(PendingIntent.
                        getActivity(ServiceB.this, 0, intent, PendingIntent.FLAG_CANCEL_CURRENT))
                .build();
        startForeground(1, notification);

### Tips:
1. startForeground让服务变成前台服务并显示通知，这时必须要setSmallIcon，否则会显示默认的通知，不显示自定义通知
2. 如何让服务一直存活 http://zhoujianghua.com/2015/07/28/black_technology_in_alipay/

# 三、Service 和 Thread

之所以有不少人会把它们联系起来，主要就是因为Service的后台概念。Thread我们大家都知道，是用于开启一个子线程，在这里去执行一些耗时操作就不会阻塞主线程的运行。而Service我们最初理解的时候，总会觉得它是用来处理一些后台任务的，一些比较耗时的操作也可以放在这里运行，这就会让人产生混淆了。但其实Service其实是运行在主线程里的，所以说service的主进程中不能执行耗时的任务，需要另外启动线程执行，例如IntentService的内部就是使用了HandlerThread的实现。

# 四、IntentService
IntentService是Service 的子类，它使用工作线程逐一处理所有启动请求。如果不要求服务同时处理多个请求，这是最好的选择。只需实现 onHandleIntent() 方法即可，该方法会接收每个启动请求的Intent，能够执行后台工作。由于大多数启动服务都不必同时处理多个请求，因此使用 IntentService 类实现服务也许是最好的选择。

IntentService的特点：
1. IntentService 会创建一个线程，来处理所有传给onStartCommand()的Intent请求
2. 创建一个请求队列，用于将 Intent 逐一传递给 onHandleIntent() 实现，不必担心多线程问题
3. 在所有的请求执行完毕后结束Service
4. 提供 onBind() 的默认实现（返回 null）
5. 提供默认的 onStartCommand() 实现，将intent传入等待队列中，然后到onHandleIntent()的实现。所以如果需要重写onStartCommand() 方法一定要调用父类的实现。

### Tips:
1. IntentService是针对StarteService设计的，由于它默认实现的onBind()方法返回值是null，所以不适合bindService()
2. 多次startService请求执行耗时任务，不会并发执行onHandleIntent()方法，而是一个一个顺序执行。当所有的任务执行完成，IntentService会自动销毁

问题：onStartCommand的返回值的具体应用场景？

Demo：

参考资料：
https://developer.android.com/guide/components/services.html
http://blog.csdn.net/guolin_blog/article/details/11952435