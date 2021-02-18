# Retrofit源码解析

# 一、基本用法

## 1.1、gradle依赖

```
implementation 'com.squareup.retrofit2:retrofit:2.9.0'
implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
```

## 1.2、定义接口

```
interface GetApi {
    /**
     * 获取用户信息
     * @return
     * @Query 注解
     */
    @GET("api/getUserInfo")
    Call<UserInfo> getUserInfo(@Query("id") List<String> userId);
 }
```

## 1.3、创建retrofit发送请求

```
Retrofit retrofit = new Retrofit.Builder().baseUrl(URL1).addConverterFactory(GsonConverterFactory.create()).build();

GetApi getApi = retrofit.create(GetApi.class);

List<String> params = new ArrayList<>();
params.add("1234");
getApi.getUserInfo(params).enqueue(new Callback<UserInfo>() {
    @Override
    public void onResponse(Call<UserInfo> call, Response<UserInfo> response) {
          String tag = response.raw().request().tag().toString();
          Log.d(TAG, "tag = " + tag);
        UserInfo userInfo = response.body();
        Log.d(TAG, "userinfo = " + userInfo + " thread " +  Thread.currentThread().getName());
    }
    @Override
    public void onFailure(Call<UserInfo> call, Throwable t) {
        Log.i(TAG, "onFailure: " + t);
    }
  });
```

# 二、注解介绍

## 2.1、请求方法

| 注解名  | 参数说明                                                     |
| ------- | ------------------------------------------------------------ |
| HTTP    | **method：**表示请求的方法GET POST等；**path ：**表示路径  ；**hasBody：**表示请求体。HTTP注解可以代替以下的所有注解。 |
| GET     | value : 通常表示对应的接口，要和baseUrl结合使用              |
| POST    | 同GET                                                        |
| PUT     | 同GET                                                        |
| PATCH   | 同GET                                                        |
| DELETE  | 同GET                                                        |
| OPTIONS | 同GET                                                        |

[HTTP请求方法](https://itbilu.com/other/relate/EkwKysXIl.html)

## 2.2、标记类

| 注解名         | 参数                                                         |
| -------------- | ------------------------------------------------------------ |
| FormUrlEncoded | 用于POST请求，表示请求体是一个Form表单，常见的网址登录就是这种类型，对应的HEAD参数是Content-Type：application/x-www-form-urlencoded |
| Multipart      | 用于POST请求，表单上传文件时使用                             |
| Streaming      | 表示响应体的数据使用流的方式返回处理，主要用于文件下载       |

[HTTP content-type](https://www.runoob.com/http/http-content-type.html)

## 2.3、参数类

### 2.3.1、注解说明

| 注解名    | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Url       | 参数里带了Url注解的会替换掉baseUrl                           |
| Path      | 替换Url中的某个路径参数                                      |
| Headers   | 多个请求头，作用于方法，**不可作为参数**                     |
| Header    | 单个请求头                                                   |
| HeaderMap | 多个请求头                                                   |
| Query     | GET请求，请求参数，**支持数组或集合**，例如 Call<UserInfo> getUserInfo(@Query("id") String[] userId); |
| QueryMap  | GET请求，多个同类型请求参数                                  |
| Field     | Post请求，请求参数，和FormUrlEncoded一起使用，**支持数组或集合** |
| FieldMap  | Post请求，多个同类型请求参数，和FormUrlEncoded一起使用       |
| Part      | 文件上传，和注解Multipart一起使用，**支持数组或集合**        |
| PartMap   | 文件上传，和注解Multipart一起使用                            |
| Body      | 多个不同类型的参数，使用对象包裹参数，包括文件上传时需要的参数，**不能和FormUrlEncoded及Multipart同时使用** |
| Tag       | 给这次请求打个Tag，https://stackoverflow.com/questions/42066885/retrofit-adding-tag-to-the-original-request-object |

### 2.3.2、文件上传

文件上传时有2种方式 RequestBody  和 MultiPart使用？？？

![image-20201208170446774](pics/image-20201208170446774.png)

![image-20201208170210061](pics/image-20201208170210061.png)



![image-20201208172412229](pics/image-20201208172412229.png)

[这是一份很详细的 Retrofit 2.0 使用教程](https://blog.csdn.net/carson_ho/article/details/73732076)

[Retrofit学习之文件和参数上传](https://www.jianshu.com/p/74b7da380855)

# 三、Retrofit构建过程

```java
Retrofit retrofit = new Retrofit.Builder().baseUrl(URL1).addConverterFactory(GsonConverterFactory.create()).build()
```

Retrofit中主要的几个对象和作用：

```java
public final class Retrofit {
  private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>(); //包含所有网络请求信息的对象

  final okhttp3.Call.Factory callFactory; //网络请求工厂
  final HttpUrl baseUrl; //网络请求的url地址
  final List<Converter.Factory> converterFactories; //数据转换器工厂的集合
  final List<CallAdapter.Factory> callAdapterFactories; //网络请求适配器工厂的集合
  final @Nullable Executor callbackExecutor; //回调方法执行器
  //...
}
```

Retrofit使用了Builder模式，通过build方法默认创建的对象。

```java
public Retrofit build() {
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");
  }

  okhttp3.Call.Factory callFactory = this.callFactory;
  if (callFactory == null) {
    // 1、callFactory 为 OkHttpClient
    callFactory = new OkHttpClient();
  }

  Executor callbackExecutor = this.callbackExecutor;
  if (callbackExecutor == null) {
    // 2、创建默认回调执行器，paltform = Android
    callbackExecutor = platform.defaultCallbackExecutor();
  }

  // Make a defensive copy of the adapters and add the default Call adapter.
  // 3、创建默认请求适配器
  List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
  callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

  // Make a defensive copy of the converters.
  // 4、创建默认的数据解析器
  List<Converter.Factory> converterFactories =
      new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

  // Add the built-in converter factory first. This prevents overriding its behavior but also
  // ensures correct behavior when using converters that consume all types.
  converterFactories.add(new BuiltInConverters());
  converterFactories.addAll(this.converterFactories);
  converterFactories.addAll(platform.defaultConverterFactories());

  return new Retrofit(
      callFactory,
      baseUrl,
      unmodifiableList(converterFactories),
      unmodifiableList(callAdapterFactories),
      callbackExecutor,
      validateEagerly);
}
```

## 3.1、回调执行器

用于回调网络请求的执行结果。

```
static final class Android extends Platform {
  Android() {
    super(Build.VERSION.SDK_INT >= 24);
  }

  @Override
  public Executor defaultCallbackExecutor() {
    return new MainThreadExecutor();
  }

  @Nullable
  @Override
  Object invokeDefaultMethod(
      Method method, Class<?> declaringClass, Object object, Object... args) throws Throwable {
    if (Build.VERSION.SDK_INT < 26) {
      throw new UnsupportedOperationException(
          "Calling default methods on API 24 and 25 is not supported");
    }
    return super.invokeDefaultMethod(method, declaringClass, object, args);
  }

  static final class MainThreadExecutor implements Executor {
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override
    public void execute(Runnable r) {
      handler.post(r);
    }
  }
}
```

## 3.2、请求适配器

默认情况下使用的是DefaultCallAdapterFactory，负责请求的执行以及回调结果。

```java
final class DefaultCallAdapterFactory extends CallAdapter.Factory {
  private final @Nullable Executor callbackExecutor;

  DefaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
	// ...
    final Type responseType = Utils.getParameterUpperBound(0, (ParameterizedType) returnType);

    final Executor executor =
        Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
            ? null
            : callbackExecutor;

    return new CallAdapter<Object, Call<?>>() {
      @Override
      public Type responseType() {
        return responseType;
      }

      @Override
      public Call<Object> adapt(Call<Object> call) {
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
      }
    };
  }
```

## 3.3、数据转换器

数据转换器可以将Response转换成我们需要的返回数据类型，默认添加的converterFactory如下：

```java
  List<Converter.Factory> converterFactories =
      new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

  // Add the built-in converter factory first. This prevents overriding its behavior but also
  // ensures correct behavior when using converters that consume all types.
  converterFactories.add(new BuiltInConverters());
  converterFactories.addAll(this.converterFactories);
  converterFactories.addAll(platform.defaultConverterFactories());
```

# 四、接口创建过程

## 4.1、创建接口代理对象

Retrofit.create，使用动态代理方式创建请求接口的代理对象，InvocationHandler 中定义了代理的规则，在调用到接口时，解析接口方法的注解参数，生成对应的ServiceMethod对象，并在调用到方法时，执行方法的参数。

```java
public <T> T create(final Class<T> service) {
  validateServiceInterface(service);
  return (T)
      Proxy.newProxyInstance(
          service.getClassLoader(),
          new Class<?>[] {service},
          new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            @Override
            public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                throws Throwable {
              // If the method is a method from Object then defer to normal invocation.
              // 1、Object 类的方法直接调用
              if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
              }
              args = args != null ? args : emptyArgs;
              // 2、平台的默认方法直接调用
              return platform.isDefaultMethod(method)
                  ? platform.invokeDefaultMethod(method, service, proxy, args)
                  : loadServiceMethod(method).invoke(args);
            }
          });
}
```

## 4.2、ServiceMethod创建

loadServiceMethod，根据方法的相关注解，生成对应的ServiceMethod对象。

```java
ServiceMethod<?> loadServiceMethod(Method method) {
  // 1、方法缓存中有直接取缓存
  ServiceMethod<?> result = serviceMethodCache.get(method);
  if (result != null) return result;

  synchronized (serviceMethodCache) {
    result = serviceMethodCache.get(method);
    if (result == null) {
      // 2、解析注解，生成对应的ServiceMethod
      result = ServiceMethod.parseAnnotations(this, method);
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```

loadService有2个入口，一个是在InvocationHandler中，另一个是在上一步的validateServiceInterface中。

若validateEagerly参数为true，那么在生成接口的动态代理对象时，解析接口中所有的方法，生成ServiceMethod对象；否则只会在调用到具体方法时才生成相关的对象。

```java
private void validateServiceInterface(Class<?> service) {
  // ...
  // 提前初始化
  if (validateEagerly) {
    Platform platform = Platform.get();
    // 解析接口中所有的方法，生成ServiceMethod对象
    for (Method method : service.getDeclaredMethods()) {
      if (!platform.isDefaultMethod(method) && !Modifier.isStatic(method.getModifiers())) {
        loadServiceMethod(method);
      }
    }
  }
}
```

## 4.3、ServiceMethod.parseAnnotations

parseAnnotations 中主要的方法有2个，作用分别是解析方法注解（包含方法参数注解）以及生成对应的请求适配器和网络数据转换器。

```java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    // 1、负责解析注解中请求时需要的相关参数
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(
          method,
          "Method return type must not include a type variable or wildcard: %s",
          returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }
    // 2、生成ServiceMethod对象，包含创建网络请求适配器和数据转换器
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```

### 4.3.1、生成RequestFactory对象

RequestFactory.parseAnnotations，通过解析方法注解和方法中的参数注解生成一个RequestFactory对象，**包含请求设定的参数**。我们先看一下RequestFactory中有哪些对象。

```java
RequestFactory(Builder builder) {
  method = builder.method;//要解析的方法
  baseUrl = builder.retrofit.baseUrl;//请求地址
  httpMethod = builder.httpMethod;//请求方式
  relativeUrl = builder.relativeUrl;//相对地址
  headers = builder.headers;//header参数
  contentType = builder.contentType;
  hasBody = builder.hasBody;
  isFormEncoded = builder.isFormEncoded;
  isMultipart = builder.isMultipart;
  parameterHandlers = builder.parameterHandlers;//方法参数解析器
  isKotlinSuspendFunction = builder.isKotlinSuspendFunction;
}
```

可以看到RequestFactory中的对象和接口的方法注解对象（Http协议参数）基本是一一对应的，RequestFactory创建时使用了Builder模式。

```java
static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
  return new Builder(retrofit, method).build();
}

RequestFactory build() {
  // 1、解析方法中的注解
  for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
  }

  if (httpMethod == null) {
    throw methodError(method, "HTTP method annotation is required (e.g., @GET, @POST, etc.).");
  }

  // Body不能和Multipart及FormUrlEncoded同时使用
  if (!hasBody) {
    if (isMultipart) {
      throw methodError(
          method,
          "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
    }
    if (isFormEncoded) {
      throw methodError(
          method,
          "FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
    }
  }
  
  // 2、解析方法中参数的注解
  int parameterCount = parameterAnnotationsArray.length;
  parameterHandlers = new ParameterHandler<?>[parameterCount];
  for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
    parameterHandlers[p] =
        parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
  }
  
  // 3、限制分成2部分的主要原因是，部分注解比如Filed是在参数中的，所以只有解析完参数后才能获取到
  // 规则1：方法注解中的相对路径和方法参数中的绝对路径不能同时为空  
  if (relativeUrl == null && !gotUrl) {
    throw methodError(method, "Missing either @%s URL or @Url parameter.", httpMethod);
  }
  // 规则2：参数注解中有body的时候（上传表单数据），方法注解中不能包含 FormEncoded，Multipart，hasbody不能为false，
  // hasbody的规则是通过请求方法来识别的，比如GET是false，POST是true
  if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
    throw methodError(method, "Non-body HTTP method cannot contain @Body.");
  }
  // 规则3：方法注解中包含FormUrlEncoded时，参数中必须包含至少一个Field注解的参数
  if (isFormEncoded && !gotField) {
    throw methodError(method, "Form-encoded method must contain at least one @Field.");
  }
  // 规则4：方法注解中包含FormUrlEncoded时，参数中必须包含Part参数
  if (isMultipart && !gotPart) {
    throw methodError(method, "Multipart method must contain at least one @Part.");
  }

  return new RequestFactory(this);
}
```

#### 1、解析**方法注解**

​      parseMethodAnnotation，**解析方法注解**，主要是请求方式，HEADER等。

```java
private void parseMethodAnnotation(Annotation annotation) {
  // 1、解析请求方法
  if (annotation instanceof DELETE) {
    parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
  } else if (annotation instanceof GET) {
    parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
  } else if (annotation instanceof HEAD) {
    parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
  } else if (annotation instanceof PATCH) {
    parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
  } else if (annotation instanceof POST) {
    parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
  } else if (annotation instanceof PUT) {
    parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
  } else if (annotation instanceof OPTIONS) {
    parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
  } else if (annotation instanceof HTTP) {
    HTTP http = (HTTP) annotation;
    parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
  } else if (annotation instanceof retrofit2.http.Headers) {
    // 2、处理HEADER
    String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
    if (headersToParse.length == 0) {
      throw methodError(method, "@Headers annotation is empty.");
    }
    headers = parseHeaders(headersToParse);
  } else if (annotation instanceof Multipart) {
    // 3、限制FormUrlEncoded 和 Multipart 同时使用
    if (isFormEncoded) {
      throw methodError(method, "Only one encoding annotation is allowed.");
    }
    isMultipart = true;
  } else if (annotation instanceof FormUrlEncoded) {
    if (isMultipart) {
      throw methodError(method, "Only one encoding annotation is allowed.");
    }
    isFormEncoded = true;
  }
}

private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      if (this.httpMethod != null) {
        throw methodError(
            method,
            "Only one HTTP method is allowed. Found: %s and %s.",
            this.httpMethod,
            httpMethod);
      }
      // 1、请求方法赋值
      this.httpMethod = httpMethod;
      this.hasBody = hasBody;

      if (value.isEmpty()) {
        return;
      }
       
      // 2、url中不能同时出现？和{}占位符
      // Get the relative URL path and existing query string, if present.
      int question = value.indexOf('?');
      if (question != -1 && question < value.length() - 1) {
        // Ensure the query string does not have any named parameters.
        String queryParams = value.substring(question + 1);
        Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
        if (queryParamMatcher.find()) {
          throw methodError(
              method,
              "URL query string \"%s\" must not have replace block. "
                  + "For dynamic query parameters use @Query.",
              queryParams);
        }
      }

      this.relativeUrl = value;
      // 3、解析path的占位符参数，在后面解析到方法参数进行匹配
      this.relativeUrlParamNames = parsePathParameters(value);
    }
```

#### 2、解析方法参数注解

​      parseParameterAnnotation, 解析方法参数注解，生成对应的ParameterHandler，在执行请求时，通过ParameterHandler将相关参数赋值到请求里。

```java
private ParameterHandler<?> parseParameterAnnotation(
    int p, Type type, Annotation[] annotations, Annotation annotation) {
  if (annotation instanceof Url) {
      //省略...
      return new ParameterHandler.RelativeUrl(method, p);
  } else if (annotation instanceof Path) {
    validateResolvableType(p, type);
    //...
    return new ParameterHandler.Path<>(method, p, name, converter, path.encoded());
  } else if (annotation instanceof Query) {
    //...  
    return new ParameterHandler.Query<>(name, converter, encoded).iterable();
  } else if (annotation instanceof QueryName) {
      //...
      return new ParameterHandler.QueryName<>(converter, encoded);
  } else if (annotation instanceof QueryMap) {
    //...
    return new ParameterHandler.QueryMap<>(
        method, p, valueConverter, ((QueryMap) annotation).encoded());
  } else if (annotation instanceof Header) {
     //...
     return new ParameterHandler.Header<>(name, converter).iterable();
  } else if (annotation instanceof HeaderMap) {
    // ...
    return new ParameterHandler.HeaderMap<>(method, p, valueConverter);
  } else if (annotation instanceof Field) {
     // ...
     return new ParameterHandler.Field<>(name, converter, encoded);
  } else if (annotation instanceof FieldMap) {
    // ...
    return new ParameterHandler.FieldMap<>(
        method, p, valueConverter, ((FieldMap) annotation).encoded());
  } else if (annotation instanceof Part) {
     // ...
     return new ParameterHandler.Part<>(method, p, headers, converter).iterable();
  } else if (annotation instanceof PartMap) {
    // ... 
    return new ParameterHandler.PartMap<>(method, p, valueConverter, partMap.encoding());
  } else if (annotation instanceof Body) {
    // ...  
    return new ParameterHandler.Body<>(method, p, converter);
  } else if (annotation instanceof Tag) {
    // ...  
    return new ParameterHandler.Tag<>(tagType);
  }

  return null; // Not a Retrofit annotation.
}
```

前面说过，Query是支持集合和数组的，具体的原理如下：

```java
// 省略... 
else if (annotation instanceof QueryName) {
  validateResolvableType(p, type);
  QueryName query = (QueryName) annotation;
  boolean encoded = query.encoded();

  Class<?> rawParameterType = Utils.getRawType(type);
  gotQueryName = true;
  // 1、集合类型
  if (Iterable.class.isAssignableFrom(rawParameterType)) {
    if (!(type instanceof ParameterizedType)) {
      throw parameterError(
          method,
          p,
          rawParameterType.getSimpleName()
              + " must include generic type (e.g., "
              + rawParameterType.getSimpleName()
              + "<String>)");
    }
    ParameterizedType parameterizedType = (ParameterizedType) type;
    Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
    // String converter
    Converter<?, String> converter = retrofit.stringConverter(iterableType, annotations);
    return new ParameterHandler.QueryName<>(converter, encoded).iterable();
    // 2、数组类型
  } else if (rawParameterType.isArray()) {
    Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
    Converter<?, String> converter =
        retrofit.stringConverter(arrayComponentType, annotations);
    return new ParameterHandler.QueryName<>(converter, encoded).array();
  } else {
    // 3、其他
    Converter<?, String> converter = retrofit.stringConverter(type, annotations);
    return new ParameterHandler.QueryName<>(converter, encoded);
  }
// 省略...
```

### 4.3.2、请求适配器 & 数据转化器 & 请求执行器

HttpServiceMethod.parseAnnotations 负责创建请求适配器 、数据转化器 、 请求执行器。

```java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
  // 1.确定接口的返回类型，这个参数很重要
  adapterType = method.getGenericReturnType();
  // 2.根据网络请求接口方法的返回类型和注解类型创建请求适配器
  CallAdapter<ResponseT, ReturnT> callAdapter =
      createCallAdapter(retrofit, method, adapterType, annotations);
  Type responseType = callAdapter.responseType();

  // 3.根据网络请求接口方法的响应类型从Retrofit对象中获取对应的数据转换器 
  Converter<ResponseBody, ResponseT> responseConverter =
      createResponseConverter(retrofit, method, responseType); 
  // 4.执行请求OkHttpClient
  okhttp3.Call.Factory callFactory = retrofit.callFactory;
 
  return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
}
```

#### 1、创建请求适配器

createCallAdapter，  默认情况下返回的是初始化时的**DefaultCallAdapterFactory**对象

```
private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
    Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
  try {
    //noinspection unchecked
    return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(method, e, "Unable to create call adapter for %s", returnType)
  }
}
```

#### 2、创建数据转换器

创建数据解析器，默认情况下使用的是传入的BuiltInConverters

```java
private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
    Retrofit retrofit, Method method, Type responseType) {
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.responseBodyConverter(responseType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(method, e, "Unable to create converter for %s", responseType);
  }
}
```

当方法返回值类型type是ResponseBody时检查一下方法是否使用了@Streaming注解标识，否则会将数据全部读取到内存中，返回ResponseBody。

```java
final class BuiltInConverters extends Converter.Factory {
  /** Not volatile because we don't mind multiple threads discovering this. */
  private boolean checkForKotlinUnit = true;

  // 1、ResponseBody转换成数据对象
  @Override
  public @Nullable Converter<ResponseBody, ?> responseBodyConverter(
      Type type, Annotation[] annotations, Retrofit retrofit) {
    if (type == ResponseBody.class) {
      return Utils.isAnnotationPresent(annotations, Streaming.class)
          ? StreamingResponseBodyConverter.INSTANCE
          : BufferingResponseBodyConverter.INSTANCE;
    }
    if (type == Void.class) {
      return VoidResponseBodyConverter.INSTANCE;
    }
    if (checkForKotlinUnit) {
      try {
        if (type == Unit.class) {
          return UnitResponseBodyConverter.INSTANCE;
        }
      } catch (NoClassDefFoundError ignored) {
        checkForKotlinUnit = false;
      }
    }
    return null;
  }

  // 2、对象转换成RequestBody对象
  @Override
  public @Nullable Converter<?, RequestBody> requestBodyConverter(
      Type type,
      Annotation[] parameterAnnotations,
      Annotation[] methodAnnotations,
      Retrofit retrofit) {
    if (RequestBody.class.isAssignableFrom(Utils.getRawType(type))) {
      return RequestBodyConverter.INSTANCE;
    }
    return null;
  }
 // ...
 }
```

#### 3、创建请求执行器

经过以上几步，我们得到一个CallAdapted对象，CallAdapted.adapt会返回一个Call对象，用于执行请求。

```java
new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
```

接口执行时先调用到，经过动态代理invoke方法调用到**ServiceMethod.invoke()**，最终会调用到callAdapter.adapt方法。

```java
  @Override
  final @Nullable ReturnT invoke(Object[] args) {
    // 传入OkHttpCall对象
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }

  static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
    private final CallAdapter<ResponseT, ReturnT> callAdapter;

    CallAdapted(
        RequestFactory requestFactory,
        okhttp3.Call.Factory callFactory,
        Converter<ResponseBody, ResponseT> responseConverter,
        CallAdapter<ResponseT, ReturnT> callAdapter) {
      super(requestFactory, callFactory, responseConverter);
      this.callAdapter = callAdapter;
    }

    @Override
    protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      return callAdapter.adapt(call);
    }
  }
```

结合DefaultCallAdapterFactory可知，默认会返回一个ExecutorCallbackCall的对象，该对象执行请求时使用的是传入的OkHttpCall。

```java
final class DefaultCallAdapterFactory extends CallAdapter.Factory {
    // 省略
    return new CallAdapter<Object, Call<?>>() {
      @Override
      public Type responseType() {
        return responseType;
      }

      @Override
      public Call<Object> adapt(Call<Object> call) {
        // adapt方法中当executor == null，就返回传入的call对象，即之前新建的OkHttpCall实例对象。
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
      }
    };
// ...    
static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }
    
    @Override
    public Response<T> execute() throws IOException {
     // 同步请求执行
      return delegate.execute();
    }
// ...
}
    
```

# 5、请求执行过程

OkHttpCall是Retrofit中封装的用于执行请求的对象，请求过程就是和OkHttp请求过程基本一致，以同步请求为例。

```java
public Response<T> execute() throws IOException {
  okhttp3.Call call;

  synchronized (this) {
    if (executed) throw new IllegalStateException("Already executed.");
    executed = true;
	// 1、创建请求
    call = getRawCall();
  }
   
  if (canceled) {
    call.cancel();
  }
  // 2、执行请求并返回结果
  return parseResponse(call.execute());
}
```

## 5.1、创建请求对象

createRawCall，创建**RealCall** 并利用前面创建的RequestFactory生成具体的request请求参数。

```java
private okhttp3.Call createRawCall() throws IOException {
  //callFactory是OkHttpClient，callFactory.newCall 
  okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}
```

RequestFactory.create的核心逻辑是利用ParameterHandler解析方法中请求参数的值，最终返回的是一个OkHttp的**Request**对象

```java
okhttp3.Request create(Object[] args) throws IOException {
  @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
  ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;
    
  int argumentCount = args.length;
    
  RequestBuilder requestBuilder =
      new RequestBuilder(
          httpMethod,
          baseUrl,
          relativeUrl,
          headers,
          contentType,
          hasBody,
          isFormEncoded,
          isMultipart);
  
  // 1、使用ParameterHandler解析请求参数
  List<Object> argumentList = new ArrayList<>(argumentCount);
  for (int p = 0; p < argumentCount; p++) {
    argumentList.add(args[p]);
    handlers[p].apply(requestBuilder, args[p]);
  }
  // 2、返回OkHttpRequest对象
  return requestBuilder.get().tag(Invocation.class, new Invocation(method, argumentList)).build();
}
```

## 5.2、处理请求结果

处理请求结果，通过responseConverter解析对象。

```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();

  // Remove the body's source (the only stateful object) so we can pass the response along.
  rawResponse =
      rawResponse
          .newBuilder()
          .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
          .build();

  // 1、请求失败的处理
  int code = rawResponse.code();
  if (code < 200 || code >= 300) {
    try {
      // Buffer the entire body to avoid future I/O.
      ResponseBody bufferedBody = Utils.buffer(rawBody);
      return Response.error(bufferedBody, rawResponse);
    } finally {
      rawBody.close();
    }
  }

  // 2、204 205 代表请求成功，但是没有资源可返回
  if (code == 204 || code == 205) {
    rawBody.close();
    return Response.success(null, rawResponse);
  }

  ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
  try {
   // 3、请求成功，解析请求返回的数据
    T body = responseConverter.convert(catchingBody);
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    // If the underlying source threw an exception, propagate that rather than indicating it was
    // a runtime exception.
    catchingBody.throwIfCaught();
    throw e;
  }
}
```

# 五、重要对象解析

## 5.1、ParameterHandler详解

#### 5.1.1、作用

ParameterHandler对象是在RequestFactory创建时生成的，参考4.3.1节；在请求Request对象创建过程中，使用ParameterHandler.apply 解析方法参数列表，不同的ParameterHandler负责对应类型参数的解析。主要的ParameterHandler类型有如下几种：

![image-20210119145830519](pics/image-20210119145830519.png)

#### 5.1.2、分析

ParameterHandler是一个抽象类，定义的抽象方法是：

```java
abstract void apply(RequestBuilder builder, @Nullable T value) throws IOException;
```

此外还有2个方法分别用于解析集合和数组类型参数，通过遍历的方式使用apply解析方法参数。

```java
abstract class ParameterHandler<T> {
  // 方法参数解析
  abstract void apply(RequestBuilder builder, @Nullable T value) throws IOException;

  // 集合类型参数ParameterHandler的解析
  final ParameterHandler<Iterable<T>> iterable() {
    return new ParameterHandler<Iterable<T>>() {
      @Override
      void apply(RequestBuilder builder, @Nullable Iterable<T> values) throws IOException {
        if (values == null) return; // Skip null values.

        for (T value : values) {
          ParameterHandler.this.apply(builder, value);
        }
      }
    };
  }
 // 数组类型多个参数的解析
  final ParameterHandler<Object> array() {
    return new ParameterHandler<Object>() {
      @Override
      void apply(RequestBuilder builder, @Nullable Object values) throws IOException {
        if (values == null) return; // Skip null values.

        for (int i = 0, size = Array.getLength(values); i < size; i++) {
          //noinspection unchecked
          ParameterHandler.this.apply(builder, (T) Array.get(values, i));
        }
      }
    };
  }
    
  static final class Header<T> extends ParameterHandler<T> {
    private final String name;
    private final Converter<T, String> valueConverter;

    Header(String name, Converter<T, String> valueConverter) {
      this.name = Objects.requireNonNull(name, "name == null");
      this.valueConverter = valueConverter;
    }

    @Override
    void apply(RequestBuilder builder, @Nullable T value) throws IOException {
      if (value == null) return; // Skip null values.
      // 1、使用对应的converter解析注解参数
      String headerValue = valueConverter.convert(value);
      if (headerValue == null) return; // Skip converted but null values.
      // 2、将参数添加到requestBuilder中
      builder.addHeader(name, headerValue);
    }
  }
 //...
 }
```

所有类型的ParameterHandler都定义在ParameterHandler中，以Header注解的解析器为例介绍下参数解析过程。

46 - 49 行，解析过程中有2个关键步骤：1、使用Converter解析注解参数，converter其实是数据转化器，也就是说converter除了可以将reponse转换成需要的对象外，还可以将输入的请求参数转换为request的参数；

2、将参数添加到RequestBuilder中。

总结一下，**ParameterHandler的作用就是将方法参数值转换为Request中相应的请求参数**。

## 5.2、请求适配器

请求适配器CallAdapter.Factory，数据转换器涉及到2个类分别是CallAdapter 和 CallAdapter.Factory，

```java
public interface CallAdapter<R, T> {
  Type responseType();

  T adapt(Call<R> call);

  /**
   * Creates {@link CallAdapter} instances based on the return type of {@linkplain
   * Retrofit#create(Class) the service interface} methods.
   */
  abstract class Factory {
    /**
     * Returns a call adapter for interface methods that return {@code returnType}, or null if it
     * cannot be handled by this factory.
     */
    public abstract @Nullable CallAdapter<?, ?> get(
        Type returnType, Annotation[] annotations, Retrofit retrofit);

    /**
     * Extract the upper bound of the generic parameter at {@code index} from {@code type}. For
     * example, index 1 of {@code Map<String, ? extends Runnable>} returns {@code Runnable}.
     */
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    /**
     * Extract the raw class type from {@code type}. For example, the type representing {@code
     * List<? extends Runnable>} returns {@code List.class}.
     */
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```

## 5.3、数据转换器

数据转换器涉及到2个类分别是Converter 和 Converter.Factory，顾名思义，Converter.Factory是converter的工厂类。常见的Converter如下：

![image-20210119151956195](pics/image-20210119151956195.png)

#### 5.3.1、分析

Converter用于构建数据转换器，主要有2个作用：1、将Response转换成我们需要的返回数据类型；2、将输入的请求参数转换为ResquestBody或String。Converter.Factory 中定义了这2种转换方式的接口：

```java
public interface Converter<F, T> {
  @Nullable
  T convert(F value) throws IOException;
    
  abstract class Factory {
      
    // 将ReponseBody转换为对象
    public @Nullable Converter<ResponseBody, ?> responseBodyConverter(
        Type type, Annotation[] annotations, Retrofit retrofit) {
      return null;
    }

     // 将对象转换为RequestBody
    public @Nullable Converter<?, RequestBody> requestBodyConverter(
        Type type,
        Annotation[] parameterAnnotations,
        Annotation[] methodAnnotations,
        Retrofit retrofit) {
      return null;
    }

    // 将对象转换为String
    public @Nullable Converter<?, String> stringConverter(
        Type type, Annotation[] annotations, Retrofit retrofit) {
      return null;
    }

    /**
     * Extract the upper bound of the generic parameter at {@code index} from {@code type}. For
     * example, index 1 of {@code Map<String, ? extends Runnable>} returns {@code Runnable}.
     */
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    /**
     * Extract the raw class type from {@code type}. For example, the type representing {@code
     * List<? extends Runnable>} returns {@code List.class}.
     */
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```

#### 5.3.2、Gson转化

以最常见的Gson数据的说明一下Converter的作用，GsonConverterFactory利用GsonResponseBodyConverter和GsonRequestBodyConverter将Gson和需要的数据类型相互转化。

```java
public final class GsonConverterFactory extends Converter.Factory {
  
  //...
  private final Gson gson;

  private GsonConverterFactory(Gson gson) {
    this.gson = gson;
  }

  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(
      Type type, Annotation[] annotations, Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);
  }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(
      Type type,
      Annotation[] parameterAnnotations,
      Annotation[] methodAnnotations,
      Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonRequestBodyConverter<>(gson, adapter);
  }
}
```

##### 1、GsonResponseBodyConverter

```java
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
  private final Gson gson;
  private final TypeAdapter<T> adapter;

  GsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
    this.gson = gson;
    this.adapter = adapter;
  }

  @Override
  public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      T result = adapter.read(jsonReader);
      if (jsonReader.peek() != JsonToken.END_DOCUMENT) {
        throw new JsonIOException("JSON document was not fully consumed.");
      }
      return result;
    } finally {
      value.close();
    }
  }
}
```

##### 2、GsonRequestBodyConverter

```java
final class GsonRequestBodyConverter<T> implements Converter<T, RequestBody> {
  private static final MediaType MEDIA_TYPE = MediaType.get("application/json; charset=UTF-8");
  private static final Charset UTF_8 = Charset.forName("UTF-8");

  private final Gson gson;
  private final TypeAdapter<T> adapter;

  GsonRequestBodyConverter(Gson gson, TypeAdapter<T> adapter) {
    this.gson = gson;
    this.adapter = adapter;
  }

  @Override
  public RequestBody convert(T value) throws IOException {
    Buffer buffer = new Buffer();
    Writer writer = new OutputStreamWriter(buffer.outputStream(), UTF_8);
    JsonWriter jsonWriter = gson.newJsonWriter(writer);
    adapter.write(jsonWriter, value);
    jsonWriter.close();
    return RequestBody.create(MEDIA_TYPE, buffer.readByteString());
  }
}
```

# 六、整体框架

![image-20210125203006574](pics/image-20210125203006574.png)

