本文主要介绍下Android Fragment的生命周期，相关的API以及和Activity通信的简单实践

# 一、生命周期：

![fragment生命周期](https://leanote.com/api/file/getImage?fileId=5936c722ab64410e79001e16)
Fragment创建销毁时，fragment和所依赖的activity生命周期的执行顺序，**注意看log的TAG**
## 1.创建时
    01-12 17:57:24.823 28028-28028/? I/ActivityLife: onCreate...
    01-12 17:57:24.843 28028-28028/? I/FragmentLife: onAttach...
    01-12 17:57:24.843 28028-28028/? I/FragmentLife: onCreate...
    01-12 17:57:24.843 28028-28028/? I/FragmentLife: onCreateView...
    01-12 17:57:24.843 28028-28028/? I/FragmentLife: onActivityCreated...
    01-12 17:57:24.843 28028-28028/? I/ActivityLife: onStart...
    01-12 17:57:24.843 28028-28028/? I/FragmentLife: onStart...
    01-12 17:57:24.843 28028-28028/? I/ActivityLife: onResume...
    01-12 17:57:24.843 28028-28028/? I/FragmentLife: onResume...
## 2.销毁时    
    01-12 17:57:40.583 28028-28028/? I/FragmentLife: onPause...
    01-12 17:57:40.583 28028-28028/? I/ActivityLife: onPause...
    01-12 17:57:40.883 28028-28028/? I/FragmentLife: onStop...
    01-12 17:57:40.883 28028-28028/? I/ActivityLife: onStop...
    01-12 17:57:40.883 28028-28028/? I/FragmentLife: onDestroyView...
    01-12 17:57:40.883 28028-28028/? I/FragmentLife: onDestroy...
    01-12 17:57:40.883 28028-28028/? I/FragmentLife: onDetach...
    01-12 17:57:40.883 28028-28028/? I/ActivityLife: onDestroy...

# 二、Activity中，Fragment的2种加载方法
## 1.静态加载
### （1）创建Fragment类和布局文件(fragment1.xml)
    public class Fragment1 extends Fragment {
        @Override
        public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, Bundle savedInstanceState) {
           return inflater.inflate(R.layout.fragment1, container, false);
        }
    }
    
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#00ff00" >
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="This is fragment 1"
            android:textColor="#000000"
            android:textSize="25sp" />
    </LinearLayout> 
### （2）在activity布局文件（activity_main.xml）中添加fragment布局
    ...
    <fragment
    android:id="@+id/fragment1"
    android:name="com.example.sven.fragementdemo.Fragment1"
    android:layout_width="0dip"
    android:layout_height="match_parent"
    android:layout_weight="1" />
    ...

## 2.动态加载
### (1) 在activity布局文件中添加FragmentLayout节点
    ...
      <FrameLayout
            android:id="@+id/fragment_container"
            android:layout_width="match_parent"
            android:layout_height="match_parent">
        </FrameLayout>
    ...
### (2) java代码中动态加载
    ...
      1.获取fragmentManager
    FragmentManager fragmentManager = getFragmentManager();
      2.获取FragmentTransaction
    FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
      3.创建需要的Fragment
    Fragment fragment = new Fragment1();
      4.动态添加fragment
        将创建的fragment添加到Activity布局文件中定义的占位符中（FrameLayout）
    fragmentTransaction.add(R.id.fragment_container,fragment).commit();
    ...
# 三、FragmentTransaction方法解析结合Fragment的生命周期

对Fragment的操作主要是通过调用FragmentTransaction类的方法进行的，FragmentTransaction的对象通常是通过getFragmentManager().beginTransaction()获取的

## 1.add & remove / replace
### (1) add 往Activity中添加一个Fragment，对应的fragment的生命周期如下：

    06-13 12:01:43.406 1986-1986/? I/Fragment3: onAttach
    06-13 12:01:43.406 1986-1986/? I/Fragment3: onCreate
    06-13 12:01:43.406 1986-1986/? I/Fragment3: onCreateView
    06-13 12:01:43.406 1986-1986/? I/Fragment3: onActivityCreated
    06-13 12:01:43.406 1986-1986/? I/Fragment3: onStart
    06-13 12:01:43.406 1986-1986/? I/Fragment3: onResume  

### (2) remove 从Activity中移除一个Fragment，和add配对使用，通常是直接用replace
    06-13 12:07:54.046 8765-8765/com.example.sven.fragementdemo I/Fragment3: onPause
    06-13 12:07:54.046 8765-8765/com.example.sven.fragementdemo I/Fragment3: onStop
    06-13 12:07:54.046 8765-8765/com.example.sven.fragementdemo I/Fragment3: onDestroyView
    06-13 12:07:54.046 8765-8765/com.example.sven.fragementdemo I/Fragment3: onDestroy
    06-13 12:07:54.046 8765-8765/com.example.sven.fragementdemo I/Fragment3: onDetach

### (3) replace 使用另一个fragment替换当前的，先remove掉当前的再add新的，参考add和remove的过程

## 2.attach & detach
### (1) attach 重建view视图，添加到UI上并显示，和detach配合使用，显示一个被detach的fragment
    06-13 11:57:46.376 29992-29992/com.example.sven.fragementdemo I/Fragment3: onCreateView
    06-13 11:57:46.376 29992-29992/com.example.sven.fragementdemo I/Fragment3: onActivityCreated
    06-13 11:57:46.376 29992-29992/com.example.sven.fragementdemo I/Fragment3: onStart
    06-13 11:57:46.376 29992-29992/com.example.sven.fragementdemo I/Fragment3: onResume
### (2) detach 会将view从UI中移除,和remove()不同,此时fragment的状态依然由FragmentManager维护，fragment的实例还存在，但是视图被销毁了
    06-13 11:58:13.486 29992-29992/com.example.sven.fragementdemo I/Fragment3: onPause
    06-13 11:58:13.486 29992-29992/com.example.sven.fragementdemo I/Fragment3: onStop
    06-13 11:58:13.486 29992-29992/com.example.sven.fragementdemo I/Fragment3: onDestroyView

## 3.hide & show
不涉及fragment生命周期，调试过程中没有看到想象中onPause、onStop、onResume函数的调用过程，hide时fragment的视图和实例都不会被销毁，只是视图可见与不可见的变化。账号登陆界面应该用的比较多，能够保存用户的输入，和detach区别就是detach不能够的保存界面的信息（例如EditText的输入），每次detach，attch时界面都会重绘
    
## 4.commit
提交对fragment的一系列操作。注意每一次对fragment的操作都要开一次事务，commit一次。commit和FragmentManager.beginTransaction()要配对使用。（很像数据库，暂时没深入研究）
    
## 5.addToBackStack(String)
把当前事务的变化情况添加到回退栈,下节详细讲下
    
## tips：
### 1.以上这些方法通常是在Activity中使用，依附于同一个Activity的多个fragment的各种切换
### 2.fragment真正的实例其实是要通过FragmentTransaction.add，只有在add时才会执行onCreate，new的时候拿到只是一个引用 并没有执行生命周期函数
### 3.使用attach/detach 或者 hide/show 都需要new出对象，并且执行add
### 4.add/remove/replace/hide/show后都要commit其效果才会在屏幕上显示出来

# 四、Fragment的回退栈

Fragment回退栈是用来保存每一次Fragment事务发生的变化，如果你将Fragment任务添加到回退栈，当点击back键时，将看到上一次的保存的Fragment，一旦Fragment完全从后退栈中弹出，用户再次点击后退键，则退出当前Activity，如果被移除的Fragment没有添加到回退栈，例如执行remove或者replace时，这个Fragment实例将会被销毁

理解以下3种情况 在同一个Actity中，fragment1 跳转到fragment2，然后按back退出时的不同情况

    public void jump2fragment2(){
        FragmentTransaction fragmentTransaction = getFragmentManager().beginTransaction();
        fragment之间跳转时
        1.只采用replace 实例会被销毁 back键一次
        fragmentTransaction.replace(R.id.fragment_container,new Fragment2());
        fragmentTransaction.addToBackStack(null);
    
        2.把当前事务的变化情况添加到回退栈,视图会被销毁但是实例还在(back 2次)
        fragmentTransaction.replace(R.id.fragment_container,new Fragment2());
        fragmentTransaction.addToBackStack(null);
    
        3.采用hide方式 实例不会被销毁,视图也不会被销毁(back 2次)
        fragmentTransaction.hide(this);
        fragmentTransaction.add(R.id.fragment_container, new Fragment2());
        fragmentTransaction.addToBackStack(null);
        fragmentTransaction.commit();
    }

# 五、Fragment与Activity之间的通信
## 1.一般Fragment依附于Activity存在，因此与Activity之间的通信可以归纳为以下3点：
### （1）Fragment中可以通过getActivity得到当前绑定的Activity的实例，然后进行操作
例如你在想在某个fragment中通过activity拿到fragment1的数据可以这样做

    public void getfragment1text() {
        getActivity().findViewById(R.id.fragment1_text)
    }
### (2) 如果你Activity中包含自己管理的Fragment的引用，可以通过引用直接访问所有的Fragment的public方法

### (3) 如果Activity中未保存任何Fragment的引用，那么没关系，每个Fragment都有一个唯一的TAG或者ID,可以通过getFragmentManager.findFragmentByTag()或者findFragmentById()获得任何Fragment实例，然后进行操作 fragment的id其实是无法设置的，add或replace时设置的id是其实要插入fragment所在ViewGroup的id，tag可以在add或replace时设置。https://stackoverflow.com/questions/9363072/android-set-fragment-id

    public class MainActivity extends AppCompatActivity {
        private static final String TAG = "ActivityLife";
        private Fragment1 fragment = new Fragment1();
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            printLog("onCreate...");
            setContentView(R.layout.activity_main);
    
            // 1.获取fragmentManager
            FragmentManager fragmentManager = getFragmentManager();
            // 2.获取FragmentTransaction
            FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
            // 3.创建需要的Fragment
            // Fragment fragment = new Fragment1();
            // 4.动态添加fragment
            // 将创建的fragment添加到Activity布局文件中定义的占位符中（FrameLayout）
            fragmentTransaction.add(R.id.fragment_container, fragment, "fragment1").commit();
        }
    
        public void getFragmentMessage(View view){
            //1.针对第2种情况
            fragment.printSth();
            //2.针对第3种情况
            Fragment fragment = getFragmentManager().findFragmentByTag("fragment1");
            printLog(fragment.toString());
            fragment.onResume();
        }
    }
    
    public class Fragment1 extends Fragment {
        ...
        public void printSth() {
            Log.i(TAG,"fragment3 printSth");
        }
    }


### （4）更复杂的用法参考：
http://blog.csdn.net/lmj623565791/article/details/37992017

## 2.应用场景主要就是通过Activity去管理多个Fragment的状态
## 思路如下：
### (1) 让各个Fragment注册一个监听接口，让Activity去implements这个监听接口

### (2) 在Activity中new出各个Fragment的对象，获取引用

### (3) 将Fragment中的事件通过listener传递到Activity中，在Fragment中通过getActivity去调用监听方法

## tips:
### Fragment中的按钮的事件响应时不能通过在配置文件中加onClick属性找到的，必须注册onClickListenr(有2种方式)
原因参考：http://blog.csdn.net/Zafir6453/article/details/51383915

## 3.以上这些自己写了个Demo，方便大家调试和学习
http://download.csdn.net/detail/time_traveller14/9870932
### 1.生命周期，回退栈，参考fragment1,fragment2,MainActivity 
### 2.FragmentTransaction及和Activity通信（fragment中不同的按钮事件响应） 参考fragment3,fragment4, Main2Activity

# 六.还未理解问题：

### 1.回退栈的使用场景，有了hide和detach为何还要加这个，仅仅是为了多按一次back键？
### 2.hide和show为何没有onPause和onResume的过程，hide为何不执行onPause，onStop呢？
### 3.remove 的真正含义销毁还是出栈，看许多资料解释是销毁掉，但为何remove掉，commit完fragment!=null，还能够通过tag获取到，但是去寻找fragment的视图元素会报错？
### 4.屏幕旋转时，已经判断了onSavedInstance==null，为何还会继续，log中 onCreate onCreateView会执行，要结合activity？

## 参考资料：
https://developer.android.com/guide/components/fragments.html
郭霖大神 http://blog.csdn.net/guolin_blog/article/details/8881711
鸿洋大神 http://blog.csdn.net/lmj623565791/article/details/37970961

### 自己的第一篇技术博客，请大家多多指教，最后不理解的问题，欢迎讨论！