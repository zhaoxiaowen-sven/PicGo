## Android原生方案的不足
我们所使用的原生路由方案一般是通过显式intent和隐式intent两种方式实现的，均存在一定意义上的缺陷：


显式intent，譬如 Intent intent = new Intent(activity, XXActivity.class); 
由于需要直接持有对应class，从而导致了强依赖关系，提高了耦合度

隐式intent，譬如 Intent it = new Intent(); 
it.setAction(“com.android.activity.MY_ACTION”); 
action等属性的定义在Manifest，导致了扩展性较差规则集中式管理，导致协作变得非常困难

原生的路由方案会出现跳转过程无法控制的问题，因为一旦使用了StartActivity()就无法插手其中任何环节了，只能交给系统管理，这就导致了在跳转失败的情况下无法降级，而是会直接抛出运营级的异常

##自定义路由框架存在的意义：
组件化：随着业务量的不断增长，app也会不断的膨胀，开发团队的规模和工作量也会逐渐增大，面对所衍生的64K问题、协作开发问题等，app一般都会走向组件化。组件化就是将APP按照一定的功能和业务拆分成多个组件module，不同的组件独立开发，组件化不仅能够提供团队的工作效率，还能够提高应用性能。而组件化的前提就是解耦，那么我们首先要做的就是解耦页面之间的依赖关系

动态跳转：在一些复杂的业务场景下（比如电商），页面跳转需要较强的灵活性，很多功能都是运营人员动态配置的，比如下发一个活动页面，我们事先并不知道具体的目标页面，期望根据下发的数据自动的选择页面并进行跳转。

## 1.配置
### 1.1 引入
    android {
        defaultConfig {
            ...
                javaCompileOptions {
                    annotationProcessorOptions {
                        arguments = [AROUTER_MODULE_NAME: project.getName()]
                    }
                }
            }
        }
    
    dependencies {
        // 替换成最新版本, 需要注意的是api
        // 要与compiler匹配使用，均使用最新版可以保证兼容
        compile 'com.alibaba:arouter-api:x.x.x'
        annotationProcessor 'com.alibaba:arouter-compiler:x.x.x'
        ...
    }
### 1.2 分包注意：    
进行module分仓的配置可参考如下方式，注意必须要在每个使用到ARouter的module的buid.gradle文件中增加
每个module都要配置的，为了应对单个module 分别编译的场景

    // app.gradle 
    annotationProcessorOptions {
        arguments = [AROUTER_MODULE_NAME: project.getName(), AROUTER_GENERATE_DOC: "enable"]
    }
        
    annotationProcessor 'com.alibaba:arouter-compiler:1.2.2'
    
    defaultConfig {
        ...
        // 用到ARouter的每个module中都必须加
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName()]
            }
        }
    }
    
    dependencies {
        //  core 中已经编译了
        implementation project(':core')
        annotationProcessor 'com.alibaba:arouter-compiler:1.2.2'
    }
    
    // core.gradle 
    dependencies {
        // gradle 3.0以上使用api 以下使用 compile 
        api 'com.alibaba:arouter-api:1.4.1'
        annotationProcessor 'com.alibaba:arouter-compiler:1.2.2'
        api 'com.alibaba:fastjson:1.1.70.android'
    }

[compile 和 implementation 的区别参考](https://blog.csdn.net/qijingwang/article/details/79805794)
### 1.3 混淆

    -keep public class com.alibaba.android.arouter.routes.**{*;}
    -keep public class com.alibaba.android.arouter.facade.**{*;}
    -keep class * implements com.alibaba.android.arouter.facade.template.ISyringe{*;}
    
    # 如果使用了 byType 的方式获取 Service，需添加下面规则，保护接口
    -keep interface * implements com.alibaba.android.arouter.facade.template.IProvider
    
    # 如果使用了 单类注入，即不定义接口实现 IProvider，需添加下面规则，保护实现
    # -keep class * implements com.alibaba.android.arouter.facade.template.IProvider

## 2.页面跳转
### 2.1 初始化sdk

    if (isDebug()) {           // 这两行必须写在init之前，否则这些配置在init过程中将无效
        ARouter.openLog();     // 打印日志
        ARouter.openDebug();   // 开启调试模式(如果在InstantRun模式下运行，必须开启调试模式！线上版本需要关闭,否则有安全风险)
    }
    ARouter.init(mApplication); // 尽可能早，推荐在Application中初始化

### 2.2 在支持路由的页面上添加注解(必选)

    // 这里的路径需要注意的是至少需要有两级，/xx/xx
    @Route(path = "/test/activity")
    public class YourActivity extend Activity {
        ...
    }
### 2.3 发起路由操作

    // 1. 应用内简单的跳转(通过URL跳转在'进阶用法'中)
    ARouter.getInstance().build("/test/activity").navigation();
    
    // 2. 跳转并携带参数
    ARouter.getInstance().build("/test/1")
                .withLong("key1", 666L)
                .withString("key3", "888")
                .withObject("key4", new Test("Jack", "Rose"))
                .navigation();

## 3.值传递和获取
### 3.1  @Autowired注解方式

    // 为每一个参数声明一个字段，并使用 @Autowired 标注, 不可是private
    // URL中不能传递Parcelable类型数据，通过ARouter api可以传递Parcelable对象
    @Route(path = "/test/activity")
    public class Test1Activity extends Activity {
        @Autowired
        public String name;
        @Autowired
        int age;
        @Autowired(name = "girl") // 通过name来映射URL中的不同参数
        boolean boy;
        @Autowired
        TestObj obj;    // 支持解析自定义对象，URL中使用json传递
    
        @Override
        protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ARouter.getInstance().inject(this);
    
        // ARouter会自动对字段进行赋值，无需主动获取
        Log.d("param", name + age + boy);
        }
    }

 ### 3.2 getIntent方式

        getIntent().getIntExtra();

### 3.3 传递对象
3.3.1 Parcelable或Serializable
        
    ARouter.getInstance().build("/test/activity").withSerializable("obj3", new TestObj3()).withParcelable("obj2", new TestObj2()).navigation()；    

3.3.2 如果需要传递自定义对象，新建一个类（并非自定义对象类），然后实现 SerializationService,并使用@Route注解标注(方便用户自行选择序列化方式)，例如：
    
    @Route(path = "/yourservicegroupname/json")
    public class JsonServiceImpl implements SerializationService {
        @Override
        public void init(Context context) {
    
        }
        // 需要引入fastJson 或者 Gson 处理序列化操作。
        @Override
        public <T> T json2Object(String text, Class<T> clazz) {
            return JSON.parseObject(text, clazz);
        }
    
        @Override
        public String object2Json(Object instance) {
            return JSON.toJSONString(instance);
        }
    }

## 4.Url跳转
### 4.1 新建一个Activity用于监听Scheme事件,之后直接把url传递给ARouter即可

    public class OpenJumpActivity extends BaseActivity {
        @Override
        protected void onCreate(@Nullable Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_open_jump);
            Uri uri = getIntent().getData();
            ARouter.getInstance().build(uri).navigation();
            finish();
        }
    }

### 4.2 AndroidManifest.xml

    <activity android:name=".activity.open.OpenJumpActivity">
        <intent-filter>
            <data
                android:host="aliyun"
                android:scheme="arouter" />
            <action android:name="arouter.demo.test" />
            <category android:name="android.intent.category.DEFAULT" />
        </intent-filter>
    </activity>

### 4.3 创建匹配的uri

    Intent intent = new Intent();
    intent.setPackage("com.vivo.test.arouterdemo");
    intent.setAction("arouter.demo.test");
    intent.setData(Uri.parse("arouter://aliyun/test/activity?name=vivo&&age=12&&girl=true"));
    startActivity(intent); 

## 5.拦截器
拦截器是自动注册的，在调用navigation时会自动的处理拦截
1.定义了拦截器一定要处理实现，没有实现的情况下，不会执行跳转等方法。
2.拦截器的priority越小优先级越高。
3.不能定义相同的priority的拦截器。

    @Interceptor(priority = 8,name = "test1Interceptor")
    public class TestInterceptor implements IInterceptor {
    
        private static final String TAG = "TestInterceptor";
    
        @Override
        public void process(Postcard postcard, InterceptorCallback callback) {
            // 必须处理
            callback.onContinue(new Postcard().withString("callback", "ok"));
        }
    
        @Override
        public void init(Context context) {
    
        }
    }

## 6.处理跳转结果（降级策略）
降级策略有2种，使用NavigationCallback或者DegradeService，自定义的方式优先于全局的方式，当设置自定义的NavigationCallback时，全局的降级策略不生效，当然自定义的还可以处理路由成功的场景等

### 1.处理单次跳转结果

    // 使用两个参数的navigation方法，可以获取单次跳转的结果
    ARouter.getInstance().build("/test/1").navigation(this, new NavigationCallback() {
        
        @Override
        public void onInterrupt(Postcard postcard) {
            Log.d(TAG, "onInterrupt: " + postcard.toString());
        }
        
        @Override
        public void onFound(Postcard postcard) {
        ...
        }
    
        @Override
        public void onLost(Postcard postcard) {
        ...
        }
    });

### 2.全局降级策略

    // 实现DegradeService接口，并加上一个Path内容任意的注解即可
    @Route(path = "/xxx/xxx")
    public class DegradeServiceImpl implements DegradeService {
    @Override
    public void onLost(Context context, Postcard postcard) {
        // do something.
    }
    
    @Override
    public void init(Context context) {
    
    }
    }

## 7.服务管理
### 7.1 暴露服务

    // 声明接口,其他组件通过接口来调用服务
    public interface HelloService extends IProvider {
        String sayHello(String name);
    }
    
    // 实现接口
    @Route(path = "/yourservicegroupname/hello", name = "测试服务")
    public class HelloServiceImpl implements HelloService {
    
        @Override
        public String sayHello(String name) {
        return "hello, " + name;
        }
    
        @Override
        public void init(Context context) {
    
        }
    }
### 7.2 发现服务
发现服务有2种方式分别是byName和byType

    public class Test {
        @Autowired
        HelloService helloService;
    
        @Autowired(name = "/yourservicegroupname/hello")
        HelloService helloService2;
    
        HelloService helloService3;
    
        HelloService helloService4;
    
        public Test() {
        ARouter.getInstance().inject(this);
        }
    
        public void testService() {
        // 1. (推荐)使用依赖注入的方式发现服务,通过注解标注字段,即可使用，无需主动获取
        // Autowired注解中标注name之后，将会使用byName的方式注入对应的字段，不设置name属性，会默认使用byType的方式发现服务(当同一接口有多个实现的时候，必须使用byName的方式发现服务)
        helloService.sayHello("Vergil");
        helloService2.sayHello("Vergil");
    
        // 2. 使用依赖查找的方式发现服务，主动去发现服务并使用，下面两种方式分别是byName和byType
        helloService3 = (HelloService) ARouter.getInstance().build("/yourservicegroupname/hello").navigation();
        helloService4 = ARouter.getInstance().navigation(HelloService.class);
        helloService3.sayHello("Vergil");
        helloService4.sayHello("Vergil");
        }
    }

## 8.跳转fragment 
跳转fragment的一般操作是先路由到Fragment的Activity，再构建出对应的fragment，对fragment进行操作。
### 8.1 创建fragement并添加注解

    @Route(path = "/test/fragment/testFragment1")
    public class TestFragment1 extends Fragment {
    
        @Nullable
        @Override
        public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
            return inflater.inflate(R.layout.activity_test_fragment, container, false);
        }
    }

### 8.2 为fragmentActivity添加路由， target 参数是对应fragment的路由路径   

    @Route(path = "/test/fragmentActivity")
    public class TestFragmentActivity extends BaseFragmentActivity {
    
        @Autowired
        String target;
    
        @Override
        protected void onCreate(@Nullable Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
    
            setContentView(R.layout.activity_fragment1);
            FragmentManager fragmentManager = getSupportFragmentManager();
    
            FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
    
            Fragment fragment = (Fragment) ARouter.getInstance().build(target).navigation();
    
            fragmentTransaction.add(R.id.fragment_container, fragment, "fragment1").commit();
        }
    }
### 8.3 发起路由
    ARouter.getInstance().build("/test/fragmentActivity").withString("target", "/test/fragment/testFragment1").navigation();

### 8.4 Fragment Issue:
https://github.com/alibaba/ARouter/issues/149

## 9.重写跳转的url

    // 实现PathReplaceService接口，并加上一个Path内容任意的注解即可
    @Route(path = "/xxx/xxx") // 必须标明注解
    public class PathReplaceServiceImpl implements PathReplaceService {
        /**
        * For normal path.
        *
        * @param path raw path
        */
        String forString(String path) {
        return path;    // 按照一定的规则处理之后返回处理后的结果
        }
    
    /**
        * For uri type.
        *
        * @param uri raw uri
        */
    Uri forUri(Uri uri) {
        return url;    // 按照一定的规则处理之后返回处理后的结果
    }
    }

## 10. 生成路由文档

    // 更新 build.gradle, 添加参数 AROUTER_GENERATE_DOC = enable
    // 生成的文档路径 : build/generated/source/apt/(debug or release)/com/alibaba/android/arouter/docs/arouter-map-of-${moduleName}.json
    android {
        defaultConfig {
            ...
            javaCompileOptions {
                annotationProcessorOptions {
                    arguments = [AROUTER_MODULE_NAME: project.getName(), AROUTER_GENERATE_DOC: "enable"]
                }
            }
        }
    }

## 参考：
https://github.com/alibaba/ARouter/blob/master/README_CN.md