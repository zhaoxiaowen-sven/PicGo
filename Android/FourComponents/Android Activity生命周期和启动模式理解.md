# Android Activity生命周期和启动模式理解

# 一、生命周期

## 1.正常情况下生命周期分析
![Activity的生命周期](https://leanote.com/api/file/getImage?fileId=594635f1ab64413c78001121)
完整生命周期：onCreate -> onDestroy
可见生命周期：onStart -> onStop
前台生命周期: onResume -> onPause
### 
## 2.常见生命周期的区别
### 1. onCreate 和 onStart
（1）可见与不可见的区别。前者不可见，后者可见。
（2）onCreate方法只在Activity创建时执行一次，而onStart方法在Activity的切换以及按Home键返回桌面再切回应用的过程中被多次调用。

### 2.onStart 和 onRestart
如果一个activity第一次创建，那么只会走onStart，如果是切换应用或者桌面，那么就会走onRestart
If it's destroyed onCreate(.) >  onStart(.) > onResume(.) is called(variables are lost, redraw).
If it's stopped onRestart(.) > onStart(.) > onResume(.) is called(variables are not lost, redraw)

### 3.onStart 和 onResume
onStart activity 可见，不在前台
onResume activity 可见，前台且可与用户交互
这2个的主要区别就是Activity此时是否可以与用户交互

### 4.onPause 和 onStop
这2个状态通常是配对执行的，
有一种情况是特殊的，新启动的Activity是一个透明的界面，那么第一个Activity执行onPause后，onStop是不会调用的
    
    <activity android:name=".SecondActivity"
        android:theme="@style/Theme.AppCompat.Dialog" />

### 5.onStop & onDestroy
onStop阶段Activity还没有被销毁，对象还在内存中，此时可以通过切换Activity再次回到该Activity，而onDestroy阶段Activity被销毁

### tips：
不同的生命周期适合不同的资源初始化或者释放工作，实践和工作中多总结。http://www.jianshu.com/p/fb44584daee3

## 3.常见操作的生命周期
### 1.Launcher点击启动
    01-16 10:57:22.429 26845-26845/? I/ActivityLife: onCreate...
    01-16 10:57:22.429 26845-26845/? I/ActivityLife: onStart...
    01-16 10:57:22.429 26845-26845/? I/ActivityLife: onResume...
### 2.back键退出
    01-16 10:58:58.139 26845-26845/? I/ActivityLife: onPause...
    01-16 10:58:58.459 26845-26845/? I/ActivityLife: onStop...
    01-16 10:58:58.459 26845-26845/? I/ActivityLife: onDestroy...
### 3.HOME键退出然后再次点击启动
    01-16 10:59:49.019 26845-26845/? I/ActivityLife: onPause...
    01-16 10:59:49.309 26845-26845/? I/ActivityLife: onStop...
    01-16 11:00:06.519 26845-26845/? I/ActivityLife: onRestart...
    01-16 11:00:06.519 26845-26845/? I/ActivityLife: onStart...
    01-16 11:00:06.519 26845-26845/? I/ActivityLife: onResume...
### 4.锁屏解锁
    01-16 11:02:33.739 26845-26845/? I/ActivityLife: onPause...
    01-16 11:02:33.779 26845-26845/? I/ActivityLife: onStop...
    01-16 11:02:35.509 26845-26845/? I/ActivityLife: onRestart...
    01-16 11:02:35.529 26845-26845/? I/ActivityLife: onStart...
    01-16 11:02:35.529 26845-26845/? I/ActivityLife: onResume...

# 2.异常情况下的生命周期分析
## 1.触发情况：
1.系统资源相关的配置发生变化导致Activity被杀死和重建
2.系统内存资源不足，lmk

## 2.执行的特殊生命周期：

    onSaveInstanceState & onRestoreInstanceState

1.onSaveInstanceState 通常在onStop之前，用于保存Activity数据 
2.onRestoreInstanceState 通常在onStart之后，恢复Activity数据 
### tips：
在onCreate和onRestoreInstanceState中都有一个参数savedInstanceState，二者的区别是：onRestoreInstanceState的参数是一定有值的，我们不用额外的判断它是否为空，但是onCreate不行，onCreate如果正常启动的话，savedInstanceState的值是null
    
## 3.典型示例，系统配置变化，屏幕旋转

    public class MainActivity extends AppCompatActivity {
    
        private static final String TAG = "ActivityLife";
    
        private static final String EDIT_TAG = "edit_tag";
        private EditText editText;
    
        @Override
        protected void onSaveInstanceState(Bundle outState) {
            Log.i(TAG, "onSaveInstanceState...");
            super.onSaveInstanceState(outState);
            String s = editText.getText().toString();
            outState.putString(EDIT_TAG,s);
        }
    
        @Override
        protected void onRestoreInstanceState(Bundle savedInstanceState) {
            super.onRestoreInstanceState(savedInstanceState);
            Log.i(TAG, "onRestoreInstanceState...");
            //onCreate 或者这里恢复数据均可
            editText.setText(savedInstanceState.getString(EDIT_TAG));
        }
    
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            Log.i(TAG, "onCreate...");
            editText = (EditText) findViewById(R.id.edit_content);
            if (savedInstanceState != null){
                Log.i(TAG, "onCreate savedInstanceState is not null");
                editText.setText(savedInstanceState.getString(EDIT_TAG));
            }
        }
    }

从Lanucher启动并且旋转一次屏幕，生命周期如下：
    
    01-16 11:59:30.459 29726-29726/? I/ActivityLife: onCreate...
    01-16 11:59:30.459 29726-29726/? I/ActivityLife: onStart...
    01-16 11:59:30.459 29726-29726/? I/ActivityLife: onResume...
    01-16 11:59:38.609 29726-29726/? I/ActivityLife: onPause...
    01-16 11:59:38.609 29726-29726/? I/ActivityLife: onSaveInstanceState...
    01-16 11:59:38.619 29726-29726/? I/ActivityLife: onStop...
    01-16 11:59:38.619 29726-29726/? I/ActivityLife: onDestroy...
    01-16 11:59:38.669 29726-29726/? I/ActivityLife: onCreate...
    01-16 11:59:38.669 29726-29726/? I/ActivityLife: onCreate savedInstanceState is not null
    01-16 11:59:38.669 29726-29726/? I/ActivityLife: onStart...
    01-16 11:59:38.669 29726-29726/? I/ActivityLife: onRestoreInstanceState...
    01-16 11:59:38.669 29726-29726/? I/ActivityLife: onResume...
## 4.onConfigurationChanged
当系统配置发生变化，如果不希望Activity重新创建，可以在AndroidManifest文件中给Activity加上属性，以屏幕旋转为例
    
    android:configChanges="orientation|screenSize"

注意：从 Android 3.2（API 级别 13）开始，当设备在纵向和横向之间切换时，“屏幕尺寸”也会发生变化。因此，在开发针对 API 级别 13 或更高版本（正如 minSdkVersion 和 targetSdkVersion 属性中所声明）的应用时，若要避免由于设备方向改变而导致运行时重启，则除了 "orientation" 值以外，您还必须添加 "screenSize" 值。 也就是说，您必须声明 android:configChanges="orientation|screenSize"。
### configChanges相关属性
![config_changes](https://leanote.com/api/file/getImage?fileId=59464bb1ab64413e9800127c)

# 二、Activity的启动模式
## 1.Activity的四种启动模式

standard:标准模式，每次启动一个Activity都会重新创建一个新的实例，不管这个实例是否已经存在
singleTop:栈顶复用模式，如果新的Activity已经位于任务栈的栈顶，那么此Activity就不会被重新创建,同时他的onNewIntent方法会被回调
singleTask:栈内复用模式，如果一个Activity在一个任务栈内存在，那么多次启动这个Activity都不会重新创建实例，它的onNewIntent方法也会被回调
singleInstance：单实例模式，具有此种模式的Activity只能单独的位于一个任务栈中
    
standard 和 singleTop都比较好理解，下面主要分析下singleTask和singleInstance的特点。会用到的调试命令：
    
    adb shell dumpsys activity | grep com.example.sven.activitydemo

## 2.singleTask的特点


​    
### 1.singleTask 和 taskAffinity 

**1.launchMode=singleTask 的应用启动时是否会创建新的任务和taskAffinity属性有关**

设置了"singleTask"启动模式的Activity，它在启动的时候，会先在系统中查找属性值affinity等于它的属值taskAffinity的任务存在；如果存在这样的任务，它就会在这个任务中启动，否则就会在新任务中启动。因此如果我们想要设置了"singleTask"启动模式的Activity在新的任务中启动，就要为它设置一个唯一的taskAffinity属性值。
    
示例：A应用中，A0（表示启动界面，A1跳转的第二个界面，依次类推）A0-A1
(1)设置SecondActivity launchMode="singleTask"，但不指定taskAffinity

    TaskRecord{576dbde #53 A=com.example.sven.activitydemo U=0 sz=2}
        Run #5: ActivityRecord{dc04be8 u0 com.example.sven.activitydemo/.SecondActivity t53}
        Run #4: ActivityRecord{bebf9d2 u0 com.example.sven.activitydemo/.MainActivity t53}

可以看到，虽然设置了singleTask的lanuchModel，但是MainActivity和SecondActivity还是在同一个任务栈中
    
(2)设置SecondActivity launchMode="singleTask" ，同时指定taskAffinity = "com.example.sven.activitydemo.second"
    
    TaskRecord{705495 #55 A=com.example.sven.activitydemo.second U=0 sz=1}
        Run #5: ActivityRecord{5a9b163 u0 com.example.sven.activitydemo/.SecondActivity t55}
    TaskRecord{9d799aa #54 A=com.example.sven.activitydemo U=0 sz=1}
        Run #4: ActivityRecord{5e9adf5 u0 com.example.sven.activitydemo/.MainActivity t54}

可以看到，设置了taskAffinity后，新的Activity就运行在新的任务栈里了
    
### tips：
默认情况下，所有Activity的所需任务栈名均为包名，所以说如果将上面的taskAffinity属性设置为包名，SecondActivity也是不会在新的任务栈中启动的

**2.在同一个任务栈中具有唯一性**
    如果设置了"singleTask"启动模式的Activity不是在新的任务中启动时，它会在已有的任务中查看是否已经存在对应的Activity实例，如果存在，就会把位于这个Activity实例上面的Activity全部结束掉，最终这个Activity实例位于任务的堆栈顶端中。
示例：
MainActity SecondActivty ThirdActivty（以上3个Activity用A、B、C代表）在2种条件下执行如下过程 A -> B ->C -> A ->B

(1) SecondActivty launchMode="singleTask"，taskAffinity=包名，最终栈中的结果如下
    
    TaskRecord{8d620aa #62 A=com.example.sven.activitydemo U=0 sz=2}
        Run #5: ActivityRecord{ab051c1 u0 com.example.sven.activitydemo/.SecondActivity t62}
        Run #4: ActivityRecord{56fcc63 u0 com.example.sven.activitydemo/.MainActivity t62}

(2) 设置SecondActivty 指定 taskAffinity = "com.example.sven.activitydemo.second"

      TaskRecord{4e4a3bc #61 A=com.example.sven.activitydemo.second U=0 sz=1}
        Run #5: ActivityRecord{f48dc2b u0 com.example.sven.activitydemo/.SecondActivity t61}
      TaskRecord{ea52f45 #60 A=com.example.sven.activitydemo U=0 sz=1}
        Run #4: ActivityRecord{9e4ab1 u0 com.example.sven.activitydemo/.MainActivity t60}

   可以看到在2种情况下， 不论是否设定taskAffinity，在SecondActivity栈顶的实例都被清掉了
    
**3.任务（Task）不仅可以跨应用（Application），还可以跨进程（Process）**

示例：
    2个应用中A,B中，2个ActivityA1,B1的singleTask的Activity的taskAffinity设置为相同，执行如下过程 A0->A1，B0->B1
    
    android:launchMode="singleTask"
    android:taskAffinity="com.example.sven.activitydemo.second"
    
    TaskRecord{a412e6e #71 A=com.example.sven.activitydemo.second U=0 sz=2}
        Run #5: ActivityRecord{12bc505 u0 com.example.sven.activitydemo2/.SecondActivityCopy t71}
      TaskRecord{fa790f #72 A=com.example.sven.activitydemo2 U=0 sz=1}
        Run #4: ActivityRecord{2610bd9 u0 com.example.sven.activitydemo2/.MainActivityCopy t72}
      TaskRecord{a412e6e #71 A=com.example.sven.activitydemo.second U=0 sz=2}
        Run #3: ActivityRecord{ca0064a u0 com.example.sven.activitydemo/.SecondActivity t71}
      TaskRecord{c224d9c #70 A=com.example.sven.activitydemo U=0 sz=1}
        Run #2: ActivityRecord{4257b8e u0 com.example.sven.activitydemo/.MainActivity t70}


可以看到相同taskAffinity的Run#5和Run#3是在同一个任务栈a412e6e中的。
    
### tips: 
Application，Task和Process的区别与联系
application翻译成中文时一般称为“应用”或“应用程序”，在android中，总体来说一个应用就是一组组件的集合。
task是在程序运行时，只针对activity的概念。说白了，task是一组相互关联的activity的集合，它是存在于framework层的一个概念，控制界面的跳转和返回。这个task存在于一个称为backstack的数据结构中，也就是说，framework是以栈的形式管理用户开启的activity。这个栈的基本行为是，当用户在多个activity之间跳转时，执行压栈操作，当用户按返回键时，执行出栈操作。
process一般翻译成进程，进程是操作系统内核中的一个概念，表示直接受内核调度的执行单位。在应用程序的角度看，我们用java编写的应用程序，运行在dalvik虚拟机中，可以认为一个运行中的dalvik虚拟机实例占有一个进程，所以，在默认情况下，一个应用程序的所有组件运行在同一个进程中。但是这种情况也有例外，即，应用程序中的不同组件可以运行在不同的进程中。只需要在manifest中用process属性指定组件所运行的进程的名字。如下所示：

     <activity
            android:name=".SecondActivityCopy"
           android:process=":remote"
            android:allowTaskReparenting="true" />

**4.allowTaskReparenting** 
    一个应用A启动另一个应用B的B1（standard模式）Activity，如果这个Activity的allowTaskReparenting属性为true的话，那么B启动后B1直接会转移到B的任务栈中。现象就是A启动B1，在启动B时，显示的就是B1，而不是B0
    
    A -> B1：
    
    TaskRecord{fc84b0d #46 A=com.example.sven.activitydemo2 U=0 sz=1}
        Run #1: ActivityRecord{8ebbcf3 u0 com.example.sven.activitydemo2/.SecondActivityCopy t46}
      TaskRecord{137b068 #45 A=com.example.sven.activitydemo U=0 sz=1}
        Run #0: ActivityRecord{b7827ad u0 com.example.sven.activitydemo/.MainActivity t45}


​    
​    
    启动B
    TaskRecord{fc84b0d #46 A=com.example.sven.activitydemo2 U=0 sz=1}
        Run #1: ActivityRecord{8ebbcf3 u0 com.example.sven.activitydemo2/.SecondActivityCopy t46}
      TaskRecord{137b068 #45 A=com.example.sven.activitydemo U=0 sz=1}
        Run #0: ActivityRecord{b7827ad u0 com.example.sven.activitydemo/.MainActivity t45}


## 4.SingleInstance的特点
**1.singleInstance模式启动的Activity具有全局唯一性**
    整个系统中只会存在一个这样的实例，如果在启动这样的Activity时，已经存在了一个实例，那么会把它所在的任务调度到前台，重用这个实例。
    
示例：
2 个应用A，B，设置**B1 launchMode="singleInstance"**， 执行如下过程，先B -> B1， 再A0 ->B1
    
    TaskRecord{9b9bb06 #108 A=com.example.sven.activitydemo2 U=0 sz=1}
        Run #3: ActivityRecord{5520232 u0 com.example.sven.activitydemo2/.SecondActivityCopy t108}
    TaskRecord{7b4a5c7 #107 A=com.example.sven.activitydemo2 U=0 sz=1}
        Run #2: ActivityRecord{6cf92f7 u0 com.example.sven.activitydemo2/.MainActivityCopy t107}
    
    TaskRecord{9b9bb06 #108 A=com.example.sven.activitydemo2 U=0 sz=1}
        Run #4: ActivityRecord{5520232 u0 com.example.sven.activitydemo2/.SecondActivityCopy t108}
    TaskRecord{3cccdb5 #109 A=com.example.sven.activitydemo U=0 sz=1}
        Run #3: ActivityRecord{facfdb u0 com.example.sven.activitydemo/.MainActivity t109}
    TaskRecord{7b4a5c7 #107 A=com.example.sven.activitydemo2 U=0 sz=1}
        Run #2: ActivityRecord{6cf92f7 u0 com.example.sven.activitydemo2/.MainActivityCopy t107}

可以看到 2 个过程中B1 的Taskrecord没有发生变化
    
**2.singleInstance模式启动的Activity具有独占性**
它会独自占用一个任务，被它开启的任何activity都会运行在其他任务中，但是否能够开启一个新任务，要看当前系统中是不是已经有了一个和要开启的Activity的taskAffinity属性相同的任务。
示例：
   B0->B1 B1->A3（B0和B1又因为上个原则，所以是在不同的任务栈中）
    
      TaskRecord{94de3cf #69 A=com.example.sven.activitydemo U=0 sz=1}
        Run #2: ActivityRecord{89ba288 u0 com.example.sven.activitydemo/.ThirdActivity t69}
      TaskRecord{2ad415c #68 A=com.example.sven.activitydemo2 U=0 sz=1}
        Run #1: ActivityRecord{73d6ca u0 com.example.sven.activitydemo2/.SecondActivityCopy t68}
      TaskRecord{6d70c65 #67 A=com.example.sven.activitydemo2 U=0 sz=1}
        Run #0: ActivityRecord{3941c44 u0 com.example.sven.activitydemo2/.MainActivityCopy t67}

先启动A0（A3和A0是运行在同一个栈中）  B0->B1 B1->A3

    TaskRecord{5db49f9 #71 A=com.example.sven.activitydemo U=0 sz=2}
      Run #3: ActivityRecord{dfea174 u0 com.example.sven.activitydemo/.ThirdActivity t71}
    TaskRecord{e7e8d3e #73 A=com.example.sven.activitydemo2 U=0 sz=1}
      Run #2: ActivityRecord{cb12857 u0 com.example.sven.activitydemo2/.SecondActivityCopy t73}
    TaskRecord{3f750ec #72 A=com.example.sven.activitydemo2 U=0 sz=1}
      Run #1: ActivityRecord{2efad8b u0 com.example.sven.activitydemo2/.MainActivityCopy t72}
    TaskRecord{5db49f9 #71 A=com.example.sven.activitydemo U=0 sz=2}
      Run #0: ActivityRecord{c39fdab u0 com.example.sven.activitydemo/.MainActivity t71}

可以看到A3（Run #2）会在A0 （Run #5）的任务栈中46bac8a
    
### tips：
启动时是否开启任务栈，考虑3个点：1.被启动的Activity的launchModel，如果是singleTask要同时考虑它的taskAffinity                
                                 2.启动Activity的launchmode(singleInstance) 
                                 3.当前是否有被启动应用的任务栈

## 4.Activity的flag

### 1.Intent.FLAG_ACTIVITY_NEW_TASK 
等价于 launchMode = singleTask

### 2.Intent.FLAG_ACTIVITY_SINGLE_TOP
等价于 launchMode = singleTop 设置singleTop 和一起使用taskAffinity ，只有singleTop的效果

### 3.Intent.FLAG_ACTIVITY_CLEAR_TOP
检查目标栈中是否有对应的实例，如果有就会将该实例之上的所有栈都清掉
    A->B（正常启动）->C->A->B(clearTop方式启动)
    (1) B的launchmode 是standard
    
    TaskRecord{fec37c7 #134 A=com.example.sven.activitydemo U=0 sz=4}
        Run #5: ActivityRecord{91e286b u0 com.example.sven.activitydemo/.MainActivity t134}
        Run #4: ActivityRecord{775d8ae u0 com.example.sven.activitydemo/.ThirdActivity t134}
        Run #3: ActivityRecord{20304d6 u0 com.example.sven.activitydemo/.SecondActivity t134}
        Run #2: ActivityRecord{b049afa u0 com.example.sven.activitydemo/.MainActivity t134}
     
    TaskRecord{fec37c7 #134 A=com.example.sven.activitydemo U=0 sz=2}
       Run #3: ActivityRecord{4fdc1db u0 com.example.sven.activitydemo/.SecondActivity t134}
        Run #2: ActivityRecord{b049afa u0 com.example.sven.activitydemo/.MainActivity t134}

可以看到，如果被启动的SecondActivity的启动模式是standard，那么B会重新创建新的实例并且之前的实例及实例之上的Activity都会出栈，recordid 发生了变化 20304d6 -> 4fdc1db

    (2) B的launchmode是singleTask
    
    TaskRecord{249ecaa #135 A=com.example.sven.activitydemo U=0 sz=4}
        Run #5: ActivityRecord{1116cb u0 com.example.sven.activitydemo/.MainActivity t135}
        Run #4: ActivityRecord{92b068e u0 com.example.sven.activitydemo/.ThirdActivity t135}
        Run #3: ActivityRecord{96bd6b6 u0 com.example.sven.activitydemo/.SecondActivity t135}
        Run #2: ActivityRecord{44bf730 u0 com.example.sven.activitydemo/.MainActivity t135}
    
    TaskRecord{249ecaa #135 A=com.example.sven.activitydemo U=0 sz=2}
        Run #3: ActivityRecord{96bd6b6 u0 com.example.sven.activitydemo/.SecondActivity t135}
        Run #2: ActivityRecord{44bf730 u0 com.example.sven.activitydemo/.MainActivity t135}

可以看到，如果被启动的SecondActivity的启动模式是SingleTask，那么B会清除实例之上的Activity，并且调用onNewIntent方法

**总结：主要2个点，清除被启动Activity之上的实例，是否创建新的实例和被启动Activity的lanuchMode有关**
    
### 4. Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS
具有这个标记的Activity不会出现在历史Activity的记录中，和android:excludeFromRecents作用相同

### tips：启动模式有2种设置方法，通过设置Intent的FLAG 和 AndroidManifest.xml ，第一种方式的优先级大于第二种方式


# 三、IntentFilter的匹配规则

Activity的显示调用和隐式调用：http://blog.csdn.net/xiao__gui/article/details/11392987

一个activity可以有多个过滤器，每个过滤器均可以有多个action，category和data

## 1.action的匹配规则

**Intent 中的 action 只要有一个与过滤器（manifest中的intent-filter）的匹配，就可调用这个过滤器所在的组件。**
    
## 2.category的匹配规则
category 表示类别，最常见就是这2个

    <category android:name="android.intent.category.LAUNCHER" />
     <category android:name="android.intent.category.DEFAULT" />

**1. 隐式启动时intent会自动的添加一个默认的category，所以隐式启动时，过滤器中默认的category必须要加**
     
     <category android:name="android.intent.category.DEFAULT"/>

**2. 隐式启动时，intent中添加的category必须包含在过滤器中**
    
## 3.data的匹配规则
**data 表示该组件可以支持的数据格式与类型，intent 至少可以匹配过滤器中的一个**

主要由两部分组成：1.mimeType 2.URI ，语法如下：

    <data android:scheme="string"
        android:host="string"
        android:port="string"
        android:path="string"
        android:pathPattern="string"
        android:pathPrefix="string"
        android:mimeType="string" />

(1) mimeType 指的是支持的数据类型与格式
    常见的有：1.text/plain 2.image/jpeg 3.video/* 4.audio/*
    / 号前面的是数据类型，后面是具体格式。

(2) URI 格式：
    
    <scheme>://<host>:<port>/[<path>]|[<pathPrefix>]|[pathPattern]

1.scheme、host、port、path分别表示URI的模式、主机名和端口号和路径
    
    例如：http://www.baidu.com:80/search/info

2.如果scheme或者host未指定那么URI就无效

3.URI 是有默认值的，content 和 file，这里不是很理解，不过发现这样一种情况：

在intent-filter 中添加任意一种mimeType，不指定URI

    <data android:mimeType="image/jpeg"/>

intent启动时，设置URI为 content:.* 或 file:.* 均可启动

    intent.setDataAndType(Uri.parse("content://abc"), "image/jpeg");

4.data 是有如下2种写法的，效果是一样的

    <data android:scheme="file"
          android:host="abc"
          android:mimeType="text/plain"/>
    
    <data android:scheme="file"/>
    <data android:host="abc"/>
    <data android:mimeType="text/plain"/>

如果一个intent-filter 中有多个data 并且他们的URI均不同，建议用第一种。
    
data 的匹配规则这里只列举的常用的一部分，还有些情况类似与如下，也可以匹配到，可以将URI 和MIMETYPE的关系理解为action和category的关系

    <data android:scheme="http" android:host="abc" android:mimeType="text/plain"/>
    <data android:scheme="http" android:host="abc2" android:mimeType="image/png"/>
    
    intent.setDataAndType(Uri.parse("http://abc"), "image/png");

只能理解为 URI 和MIMETYPE 其实类似于 action 和 category的关系，采用如下的写法更能表现两者的关系，具体大家在实践中多多理解

      <data android:scheme="file" android:host="abc" />
      <data android:scheme="file" android:host="abc2" />
      <data android:mimeType="text/plain" />
      <data android:mimeType="image/png" />

### tips:
1.如果要为Intent指定完整的data，必须要调用setDataAndType方法，不能先调用setData然后调用setType，因为这两个方法会彼此清除对方的值。
2.当我们启动Activity可以通过resolveActivity判断是否有对应的Activity
    
    if(this.getPackageManager().resolveActivity(intent, 0) != null){
        startActivity(intent);
    }

3.特殊的一组action和category，标记应用启动时打开的Activity，少了任何一个都没有意义

    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>

4.显示Component调用也是可以跨应用的，需要显示指定Activity属性 android:exported="true" 或者添加一个intent-Filter，否则会报AndroidRuntime异常，通过Tip 2是不能检查出intent是否正常。

以上这些自己写了个Demo，方便大家调试和学习：


**问题：**
1. onSaveInstanceState 为何会在跳转到别的Activity、Home键以及锁屏时调用？
2. category 和 data 的更多用法？

**参考资料：**
https://developer.android.com/reference/android/app/Activity.html
https://developer.android.com/guide/topics/manifest/activity-element.html?hl=zh-cn#config
http://blog.csdn.net/u011240877/article/details/71305797
《Android 开发艺术之旅》