# BroadcastReceiver 基础知识

BroadcastReceiver 四大组件之一，主要用于监听手机状态的变以及不同组件和应用间的通信。

## 一.注册广播
### 1.静态注册：
1. 在AndroidMainfest中添加receiver

        <receiver android:name=".MyReceiver">
            <intent-filter>
                <action android:name="android.intent.action.myreceiver" />
            </intent-filter>
        </receiver>

2. 创建类继承BroadcastReceiver的，并且复写onReceive()方法

        public class MyReceiver extends BroadcastReceiver {
            @Override
            public void onReceive(Context context, Intent intent) {
                Log.i(TAG, "receive action ");
            }
        }

### 2. 动态注册

1. 创建广播类和实例
2. 创建intentFilter并且添加需要接收的action
3. 调用registerReceiver方法注册
4. 可以通过unregisterReceiver取消

        public class MainActivity extends Activity {
            private BroadcastReceiver mBroadcastReceiver;
        
            @Override
            protected void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);
                setContentView(R.layout.activity_main);
        
                mBroadcastReceiver = new MyBroadcastReceiver();
                IntentFilter intentFilter = new IntentFilter();
                intentFilter.addAction("MyIntent");
                registerReceiver(mBroadcastReceiver, intentFilter);
            }
            @Override
            protected void onDestroy() {
                super.onDestroy();
                unregisterReceiver(mBroadcastReceiver);
            }
        }

###Tips：
1. action的注册和解注册成对出现在context对应的生命周期中，例如onCreate和onDestroy，以及onResume和onPause
2. 同一个receiver可以同时接收动态注册和静态注册的广播
3. 动态广播在应用没有启动时，是无法接收到的，即使加了Intent.FLAG_INCLUDE_STOPPED_PACKAGES（测试失败）

## 二、广播的类型
###1. 系统广播
   系统广播，当手机状态发生变化时，都会发出相应的系统广播。如：网络状态，解锁等。注意：有些系统广播必须态注册才有效：SCREEN_ON，SCREEN_OFF
###2. 普通广播
   自定义的广播，通常用于应用内或应用间通信。
###3. 有序广播
1. 多个具当前已经注册且有效的BroadcastReceiver接收有序广播时，是按照先后顺序接收的，先后顺序判定标准为：将当前系统中所有有效的动态注册和静态注册的BroadcastReceiver按照priority属性值从大到小排序，对于具有相同的priority的动态广播和静态广播，动态广播会排在前面。
   
        <receiver android:name=".OrderedReceiver1">
            <intent-filter android:priority="100">
                <action android:name="com.sven.action.my.receiver.orderd.receiver" />
            </intent-filter>
        </receiver>
        <receiver android:name=".OrderedReceiver2">
            <intent-filter android:priority="1000">
                <action android:name="com.sven.action.my.receiver.orderd.receiver" />
            </intent-filter>
        </receiver>

 当然动态注册时也可以设置优先级：

        intentFilter.setPriority(1000);

2. 先接收的BroadcastReceiver可以对此有序广播进行截断，使后面的BroadcastReceiver不再接收到此广播，
  
        public class OrderedReceiver2 extends BroadcastReceiver {
            private static final String TAG = "OrderedReceiver2";
            @Override
            public void onReceive(Context context, Intent intent) {
                Log.i(TAG, "action = "+intent.getAction());
                abortBroadcast();
            }
        }

3. receiver之间的通信
优先接收到Broadcast的Receiver可通过setResultExtras(Bundle)方法将处理结果存入Broadcast中，下一个Receiver 可通过getResultExtras(true)方法获取上一个 Receiver传来的数据。

        //修改
        public class OrderedReceiver2 extends BroadcastReceiver {
            private static final String TAG = "OrderedReceiver2";
            @Override
            public void onReceive(Context context, Intent intent) {
                Log.i(TAG, "action = "+intent.getAction());
                Bundle b = new Bundle();
                b.putInt("value", 101);
                setResultExtras(b);
            }
        }
        
        //接收    
        public class OrderedReceiver1 extends BroadcastReceiver {
            private static final String TAG = "OrderedReceiver1";
            @Override
            public void onReceive(Context context, Intent intent) {
                Log.i(TAG, "action = "+intent.getAction());
                Bundle b = getResultExtras(true);
                Log.i(TAG,"value = "+b.get("value"));
            }
        }


##三、广播和权限
### 1. 谁有权限接收 (和安装的顺序有关 需要先装发送者)

1. 发送者的manifest声明权限
   
        <permission android:name="com.sven.permission.my.receiver.RECEIVE"/>
    
2. 发送时添加权限
        
        public void sendBroadcast(View view) {
            sendBroadcast(new Intent("com.sven.action.my.receiver"),
                    "com.sven.permission.my.receiver.RECEIVE");
        }
    
3. 接收app的manifest要添加对应的权限
   
        <uses-permission android:name="com.sven.permission.my.receiver.RECEIVE"/>
    
    
### 2. 谁有权限发送 （和安装顺序有关，先安装接收者） 
1. 接收者的manifest文件中声明权限

           <permission android:name="com.sven.permission.my.receiver.SEND"/>
    
2. 接收者的receiver节点中添加
   
            <receiver
                android:name=".MyReceiverWithPermission"
                android:permission="com.sven.permission.my.receiver.SEND">
                <intent-filter>
                ....
                </intent-filter>
            </receiver>

3. 发送者的manifest文件中使用

        <uses-permission android:name="com.sven.permission.my.receiver.SEND"/>

### Tips：
自定义的权限最好在2个app中同时声明，（测试出现了比较奇怪的情况，声明权限的app，必须先安装，否则还是会报权限问题），使用系统权限不会不存在以上问题

### 四、安全高效地使用广播的一些原则：

1. 如果不需要发送到应用外，同一个应用内的广播尽量使用LocalBroadcastManager
2. 尽量使用动态注册的广播，而且有些系统广播，比如说 CONNECTIVITY_ACTION 在7.0之后只能通过动态注册接收
3. 发送广播时明确广播的接受者：
    (1)发送时添加权限
    (2)通过setPackage 指定应用
    (3)使用LocalBroadcastManager
4. 广播的命名尽量保证唯一
5. 注册一个广播时，限制广播的接收者：

    (1)添加一个权限, mainfest 中指定，动态注册可以使用该接口添加权限
        
        public abstract Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter, String broadcastPermission, Handler scheduler);
    
    (2)仅仅只是应用内使用时，可以使用设置 android:exported="false"。这样就不会接收应用外的广播了
    (3)使用LocalBroadcastManager
6. onReceive方法运行在主线程中，所以不能执行耗时任务
7. 不要通过广播启动activity，可以使用通知替代

五、onReceive
系统执行onReceive方法时，receiver会被当成前台进程，不会被杀，但是当onReceive()方法返回后，会被当成一个低优先级的程序，很容易被系统杀掉，在onReceive执行耗时异步任务时，最好通过goAsync防止执行过程中被系统杀掉。
onReceive 的context是Application的context？？？

    public class MyBroadcastReceiver extends BroadcastReceiver {
        private static final String TAG = "MyBroadcastReceiver";
        @Override
        public void onReceive(final Context context, final Intent intent) {
            final PendingResult pendingResult = goAsync();
            AsyncTask<String, Integer, String> asyncTask = new AsyncTask<String, Integer, String>() {
                @Override
                protected String doInBackground(String... params) {
                    StringBuilder sb = new StringBuilder();
                    sb.append("Action: " + intent.getAction() + "\n");
                    sb.append("URI: " + intent.toUri(Intent.URI_INTENT_SCHEME).toString() + "\n");
                    Log.d(TAG, log);
                    // Must call finish() so the BroadcastReceiver can be recycled.
                    pendingResult.finish();
                    return data;
                }
            };
            asyncTask.execute(); 
    }
}
###Tips:
可以在通过一下这种方法让onReceive执行在handlerThread中。

    private Handler handler; // Handler for the separate Thread
    HandlerThread handlerThread = new HandlerThread("MyNewThread");
    handlerThread.start();
    // Now get the Looper from the HandlerThread so that we can create a Handler that is  attached to the HandlerThread
    // NOTE: This call will block until the HandlerThread gets control and initializes its Looper
    Looper looper = handlerThread.getLooper();
    // Create a handler for the service
    handler = new Handler(looper);
    // Register the broadcast receiver to run on the separate Thread
    registerReceiver (myReceiver, intentFilter, broadcastPermission, handler);

https://stackoverflow.com/questions/10682241/register-a-broadcast-receiver-from-a-service-in-a-new-thread
## 问题：
1. 有哪些静态广播可以启动进程，是否有其他要求？
2. 上文提到的发送及接收时permission的问题？
3. BroadcastReceiver 耗时任务通过开启service执行和异步执行各有什么应用场景？

参考资料：
https://developer.android.com/guide/components/broadcasts.html#security_considerations_and_best_practices
http://www.cnblogs.com/lwbqqyumidi/p/4168017.html

Demo：
http://download.csdn.net/detail/time_traveller14/9892205
http://download.csdn.net/detail/time_traveller14/9892207