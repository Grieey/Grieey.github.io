---
title: Retrofit学习总结
date: 2020-04-14T18:31:23+08:00
tags: 
    - Retrofit
    - 源码学习
categories : 
    - Android
    - 学习记录

---

## 优势

**Retrofit2** 是一款开源的网络请求框架，它对Android中的网络请求从请求的构建，请求线程的切换和请求的调度以及对请求结果的解析等一系列的流程进行了统一封装，使得我们只需要将一次网络请求的关注点从请求的具体细节和管理中转移到只需要关注请求的配置，极大的提高了网路请求的操作，而且对接口的管理也变得十分容易，简单易用易管理。

第二是整个框架的扩展性很强，对于不同的项目，请求的底层发起或者是请求的解析及请求的回调等等实现可能有差异，在Retrofit中都将上诉的每一个过程以配置项的形式配置给Retrofit类，从而可以很轻易的实现替换或者自定义。

第三是在整个请求的过程中，在Retrofit的内部都进行了大量的验证和检验，这使得我们在是使用的过程中能比较轻易的发现配置的错误或者处理请求失败等情况。

## 原理及流程分析

**Retrofit2**的大概原理是利用动态代理来实现的，简单说就是将我们配置好的接口类，在程序运行时通过动态代理生成实际的代理类，然后由代理具体的去生成请求需要的Call对象。这个Call对象会具体的调用进行请求的异步接口`enqueue`或者是同步接口`execute`，来触发实际的请求操作。

下面是[stay](https://www.jianshu.com/p/45cb536be2f4)所绘制的一张流程图

![](https://upload-images.jianshu.io/upload_images/625299-29a632638d9f518f.png)



从上图不仅仅可以很清晰的看出一个请求的大概流程，还能知道整个Retrofit的大概结构是怎么样的。接下来就结合上图和具体的源码来看看在Retrofit中上图的细节是如何实现的。

### 建立请求的配置

```kotlin
//创建Retrofit对象，并对其进行配置
val retrofit: Retrofit = Retrofit.Builder()
      .baseUrl(BASE_URL)
      .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
      .client(client)
      .addConverterFactory(GsonConverterFactory.create())
      .build()
```

构建`Retrofit`使用了构建者模式，这个模式可以很好的解构构建的条件和需要构建的对象，让我们可以根据自己需要来构建不同的请求。如上除了配置`baseUrl`，还添加了`CallAdapterFactory`和`GsonConverterFactory`两个对象，一个用于发起请求一个用于解析请求。

### 生成Call对象

```kotlin
val service: GitHubService = retrofit.create(GitHubService::class.java)
val call = service.listRepo("xxx")

interface GitHubService {
  @GET("users/{user}/repos")
  fun listRepos(@Path("user") user: String?): Call<List<Repo?>?>?
}
```

这里的`GitHubService`就是我们自己定义的Api的接口类，里面生命了业务中需要使用的Api。接下里的重点是`create`这个方法

```java
public <T> T create(final Class<T> service) {
    // 1.验证接口 
    validateServiceInterface(service);
    // 2.返回动态代理的对象
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
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                return platform.isDefaultMethod(method)
                    ? platform.invokeDefaultMethod(method, service, proxy, args)
                    : loadServiceMethod(method).invoke(args);
              }
            });
  }
```

先看第一步，验证接口，代码如下，代码的逻辑主要是两块。

先是遍历的验证，如果接口是泛型接口就会直接抛出异常。然后第二个是一个提前验证和缓存，如果`validateEagerly`这个变量我们设置为true，这个时候就会先调用`loadServiceMethod`这个方法，从上面的代码我们也可以看到，这个方法其实也是动态代理最后调用的方法，那么这里先调用的目的就是为了验证接口是否合法从而提前暴露出问题，但是由于该方法里面是反射来进行的，接口一次性都加载会比较耗费性能，一般都是调试时打开，正式环境会关闭。

```java
 private void validateServiceInterface(Class<?> service) {
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }

    Deque<Class<?>> check = new ArrayDeque<>(1);
    check.add(service);
    while (!check.isEmpty()) {
      Class<?> candidate = check.removeFirst();
      // 验证是否是泛型接口
      if (candidate.getTypeParameters().length != 0) {
        StringBuilder message =
            new StringBuilder("Type parameters are unsupported on ").append(candidate.getName());
        if (candidate != service) {
          message.append(" which is an interface of ").append(service.getName());
        }
        throw new IllegalArgumentException(message.toString());
      }
      Collections.addAll(check, candidate.getInterfaces());
    }

    if (validateEagerly) {
      Platform platform = Platform.get();
      for (Method method : service.getDeclaredMethods()) {
        if (!platform.isDefaultMethod(method) && !Modifier.isStatic(method.getModifiers())) {
          loadServiceMethod(method);
        }
      }
    }
  }
```

接下里就是第二步，第二步调用了`Proxy.newProxyInstance`方法来生成动态代理类，这个方法会接收三个参数，第一个是一个`classLoader`，是**JVM**中用于生成代理类的，第二个就是需要代理的接口的数组，第三个是一个`InvocationHandler`的匿名类，这个中就是我们动态代理类最后实际生成的代理类会执行的代码。例如我们的接口如下声明

```kotlin
val call = service.listRepo("xxx")
```

这里的`service`是我们声明的接口类，那它的具体实现类就是上面的代理所生成的，大概会如下伪代码

```kotlin
class xxPro : GitHubService {
  val handler: InvocationHandler
 override fun listRepos(user: String?): Call? {
    val method = getMethod(""listRepos"") 
    return handler.invoke(this, method, user )
  }
}
```

在代理中会持有我们传入的那个`InvocationHandler`匿名类的对象，然后调用其`invoke`方法，这个方法正好就是上面`create`中所实现的逻辑。这一段就实现了将我们配置的接口进行了统一的封装处理，使得我们只需要关注接口的配置即可。回到上面的代码，在`invoke`内部也有两个处理，一个是判断了调用的方法是否是`Object`对象的方法，因为所有的对象都是继承自`Object`对象，所以如果调用的方法是`Object`的就直接执行，这里还判断了是否是`platform.isDefaultMethod(method)`，以下是该方法的实现，这个方法就是判断的是否是**Java8**的默认方法(关于啥是默认方法，[这里有解释](https://www.runoob.com/java/java8-default-methods.html))，如果是也直接执行即可。

```java
  boolean isDefaultMethod(Method method) {
    return hasJava8Types && method.isDefault();
  }
```

最后就是调用的一开始验证准备调用的那个`loadServiceMethod` 来获取一个 `ServiceMethod`对象并调用其`invoke`方法，我们来看看实现

```java
  ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

这个方法，其实就做了两件事，一是建立`ServiceMethod`对象，而是建立缓存。所以我们直接看`parseAnnotations`方法就可以了。

```java
abstract class ServiceMethod<T> {
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
	...非核心逻辑...
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }

  abstract @Nullable T invoke(Object[] args);
}
```

这是个抽象方法，但是它其实只有一个实现类，那就是`HttpServiceMethod ,然后看看具体的实现

```java
 static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
   ...
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
   ...
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    if (!isKotlinSuspendFunction) {
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForResponse<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
    } else {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForBody<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
              continuationBodyNullable);
    }
  }
```

这个方法精简一下就是上面的样子了，依次创建了`CallAdapter、responseConverter、callFactory`这几个对象，然后根据条件去创建`CallAdapted`(**isKotlinSuspendFunction**是指是否是Kotlin中的`suspend`关键字修饰的方法，目前一般都不是，如果有用到协程可以去看看下面的实现，看看有啥区别)。这里的`CallAdapted`是`HttpServiceMethod`的子类。最后我们就是看看它的`invoke`方法的实现

```java
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }
```

这里创建了一个`OkhttpCall`，然后调用`adapt`方法。而`adapt`方法本身是一个抽象方法，它的具体实现是在刚刚创建的那个`CallAdapted`类中

```java
protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      return callAdapter.adapt(call);
    }
```

看到这里其实也能猜到大概的功能了，调用`callAdapter`无非就是对`call`对象做了一层适配的转换，所以最主要的还是那个`call`对象。

那么到这里，这一段开头的`create`方法创建`call`对象的流程算是完成了。

### 请求的发起和调度

```kotlin
call?.enqueue(object : Callback {
      override fun onFailure(call: Call, e: IOException) {
      }

      override fun onResponse(call: Call, response: Response) {
      }

    })
```

这是一个`call`进行异步请求的代码，调用了`enqueue`方法，通过上面的分析我们知道，在`retrofit`内部实际是生成的一个`OkhttpCall`，所以接下来就是去分析它了。

```java
  public void enqueue(final Callback<T> callback) {
	...
    okhttp3.Call call;
	...

    synchronized (this) {
	...
    call.enqueue(
        new okhttp3.Callback() {
          @Override
          public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
            Response<T> response;
            try {
              response = parseResponse(rawResponse);
            } catch (Throwable e) {
              throwIfFatal(e);
              callFailure(e);
              return;
            }

            try {
              callback.onResponse(OkHttpCall.this, response);
            } catch (Throwable t) {
              throwIfFatal(t);
              t.printStackTrace(); // TODO this is not great
            }
          }

          @Override
          public void onFailure(okhttp3.Call call, IOException e) {
            callFailure(e);
          }

          private void callFailure(Throwable e) {
            try {
              callback.onFailure(OkHttpCall.this, e);
            } catch (Throwable t) {
              throwIfFatal(t);
              t.printStackTrace(); // TODO this is not great
            }
          }
        });
  }
```

在`OkhttpCall`的`enqueue`方法中，调用了`Okhttp3`的`call`(注意这里，**OkHttpCall**是**Retrofit**中定义的对象，而**Okhttp3.call**是**OkHttp3**中定义的对象)。这里就是进行实际的请求了，那么线程的切换是在哪儿进行的呢？回想下上面我们最后最终的代码进行一次猜想，那就是最后调用的`callAdapter.adapt`这个方法，代码我们就没有跟下去了，所以我回头去看了看这个`callAdapter`的生成，发现了如下代码(在Retrofit中)

```java
List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
```

也就是`callAdapter`是从这个`defaultCallAdapterFactories`中产生的，这里还接收了一个`callbackExector`的参数，从命名可以看出这个对象应该是一个`ExecutorService`对象（从代码上也能印证这一点）。而这个`Factory`中的`adapt`方法是把`OkhttpCall`封装为了`ExecutorCallbackCall`。

```java
      public Call<Object> adapt(Call<Object> call) {
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
      }
```

这个call的内部接管了`enqueue`方法的调用，调用了`callbackExecutor.execute`的方法，从而实现了线程的切换。而这个`delegate`就是刚刚传入的`OkhttpCall`对象。

```java
    public void enqueue(final Callback<T> callback) {
      Objects.requireNonNull(callback, "callback == null");

      delegate.enqueue(
          new Callback<T>() {
            @Override
            public void onResponse(Call<T> call, final Response<T> response) {
              callbackExecutor.execute(
                  () -> {
                    if (delegate.isCanceled()) {
                      // Emulate OkHttp's behavior of throwing/delivering an IOException on
                      // cancellation.
                      callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                    } else {
                      callback.onResponse(ExecutorCallbackCall.this, response);
                    }
                  });
            }

            @Override
            public void onFailure(Call<T> call, final Throwable t) {
              callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
            }
          });
    }
```

请求是在子线程，通过代码测试，发现最后的`Response`回调中，是在主线程。所以我们回去印证下我们的猜想，发现这个`executor`的具体实现如下，它是一个在主线程的`handler`。

```java
static final class MainThreadExecutor implements Executor {
  private final Handler handler = new Handler(Looper.getMainLooper());

  @Override
  public void execute(Runnable r) {
    handler.post(r);
  }
}
```

