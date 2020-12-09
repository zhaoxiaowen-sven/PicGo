# Retrofit

# 1、基本用法

  

# 2、注解介绍

## 2.1、请求

#### 1、请求方法

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

#### 2、标记类

| 注解名         | 参数                                                         |
| -------------- | ------------------------------------------------------------ |
| FormUrlEncoded | 用于POST请求，表示请求体是一个Form表单，常见的网址登录就是这种类型，对应的HEAD参数是Content-Type：application/x-www-form-urlencoded |
| Multipart      | 用于POST请求，表单上传文件时使用                             |
| Streaming      | 表示响应体的数据使用流的方式返回处理，主要用于文件下载       |

[HTTP content-type](https://www.runoob.com/http/http-content-type.html)

#### 3、参数类

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

文件上传时有2种方式 RequestBody  和 MultiPart使用？？？

![image-20201208170446774](pics/image-20201208170446774.png)

![image-20201208170210061](pics/image-20201208170210061.png)



![image-20201208172412229](pics/image-20201208172412229.png)

[这是一份很详细的 Retrofit 2.0 使用教程](https://blog.csdn.net/carson_ho/article/details/73732076)

[Retrofit学习之文件和参数上传](https://www.jianshu.com/p/74b7da380855)

# 3、请求创建流程

1、使用动态代理方式创建一个请求接口代理对象

​       InvocationHandler 中定义了代理的规则，**在调用到方法时**，解析方法的注解中的参数，生成对应的ServiceMethod对象，再执行方法的参数。

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

2、loadServiceMethod

​      loadServiceMethod的主要作用是根据相关注解，生成对应的ServiceMethod对象。

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

 若validateEagerly参数为true，那么在生成动态代理对象时，解析接口中所有的方法，生成ServiceMethod对象。

```java
private void validateServiceInterface(Class<?> service) {
  ...
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

3、ServiceMethod.parseAnnotations

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
    // 2、生成ServiceMethod对象
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```

parseAnnotations 中主要的方法有2个，先来看第一个。

4、RequestFactory.parseAnnotations

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
  //规则4：方法注解中包含FormUrlEncoded时，参数中必须包含Part参数
  if (isMultipart && !gotPart) {
    throw methodError(method, "Multipart method must contain at least one @Part.");
  }

  return new RequestFactory(this);
}
```

### 4.1 parseMethodAnnotation

​       解析方法上的注解，主要是请求方式，HEADER等，这里可以确定哪些注解是可以作用到方法上的。

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

### 4.2 parseParameterAnnotation

​         根据不同的注解类型，生成对应的ParameterHandler，通过ParameterHandler将相关参数赋值到请求里。

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
    Converter<?, String> converter = retrofit.stringConverter(type, annotations);
    return new ParameterHandler.QueryName<>(converter, encoded);
  }
// 省略...
```



# 5、Converter.Factory



# 6、CallAdapter.Factory