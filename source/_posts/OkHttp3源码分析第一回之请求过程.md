---
title: OkHttp3源码分析第一回之请求过程
categories: "源码分析"
---



有关Android网络请求的开源库有很多，而OkHttp无疑是最优秀的网络请求库，它几乎能高效完美处理各种复杂的Http请求。说实话，这个库还是很值得去阅读它的源码的。所以我们今天来分析一下它的源码吧。本文基于OkHttp版本3.11.0。部分代码使用Kotlin语言来编写。

# 一. OkHttp执行请求

首先我们来构建两次（一个同步，一个异步）简单的请求：

```kotlin
fun testOkHttp(url: String) {
        //配置OkHttpClient所需的一些参数到一个Builder里面
        val builder = OkHttpClient.Builder()
                .connectTimeout(15, TimeUnit.SECONDS)
                .readTimeout(15, TimeUnit.SECONDS)
                .writeTimeout(15, TimeUnit.SECONDS)
        //然后用该builder创建一个OkHttpClient
        val client = builder.build()
        //构建一个请求
        val request = Request.Builder()
                .url(url)
                .build()
        //1.执行同步请求，请求执行和返回结果在同一个线程
        //同步请求结果
        val responseExecute: Response? = client.newCall(request).execute()
        //2.执行异步请求，请求执行和返回结果不在同一个线程，返回结果以回调的方式
        client.newCall(request).enqueue(object : Callback {
            override fun onFailure(call: Call?, e: IOException?) {
            }

            override fun onResponse(call: Call?, response: Response?) {
                //异步请求结果
                val responseEnqueue: Response? = response
            }
        })
    }
```

以上代码，我们揪出几个点，然后逐一击破来了解OkHttp请求的整个过程。

- OkHttpClient构建
- newCall方法
- Call接口和它的子类RealCall
- 同步请求execute()
- 异步请求enqueue()
- Dispatcher调度

# 二. 源码分析

## 1. OkHttpClient构建

我们先看OkHttpClient类，该类提供了两个构造器:

```java
public OkHttpClient() {
    this(new Builder());
  }
```

我们可以直接使用OkHttpClient构造器来创建一个实例，它会直接调用兄弟构造器：

```java
OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    this.proxy = builder.proxy;
    ...
}
```

它使用的的参数是Builder里面默认设置的参数：

```java
public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      if (proxySelector == null) {
        proxySelector = new NullProxySelector();
      }
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
    }
```

当然了，我们也可以不采用默认的Builder参数，像开头代码示例一样先构建一个Builder，我们在Builder里面自由设置自己的配置参数，然后再build出一个OkHttpClient实例：

```java
public OkHttpClient build() {
      return new OkHttpClient(this);
    }
```

> 我们可以这么理解，OkHttpClient里需要的配置太多了，采用[建造者设计模式](https://github.com/simple-android-framework/android_design_patterns_analysis/tree/master/builder/mr.simple)可以让调用者灵活配置一个自己的OkHttpClient。

## 2. newCall方法

我们先看newCall方法：

```java
@Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```

这是一个重写了Call接口的内部接口Factory的call方法，然后调用的是RealCall的newRealCall方法。RealCall类实现了Call接口，newRealCall方法返回了一个Call类型的实例。所以到目前为止，我们知道同步请求execute()和异步请求enqueue()实际是在Call接口的子类RealCall中执行的。那我们接下来重点来了解一下这个Call接口和它的子类RealCall。

## 3. Call接口和它的子类RealCall

```java
public interface Call extends Cloneable {
    ...
    //同步请求，直接返回Response
    Response execute() throws IOException;
    //异步请求，通过回调返回Response
    void enqueue(Callback responseCallback);
    ...
    //OkHttpClient中的newCall()方法是继承这个内部接口来实现的
    interface Factory {
    	Call newCall(Request request);
  	}
}
```

可以看到Call接口向外部暴露了一个工厂接口来"生产"自己，是不是很熟悉的[工厂设计模式](https://baike.baidu.com/item/%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F/9852061?fr=aladdin)中的工厂接口设计。Call接口只有一个子类RealCall类，我们直接调到RealCall类中来看重头戏execute()和enqueue()。

## 4. 同步请求execute()

直接上源码分析：

```java
@Override public Response execute() throws IOException {
    synchronized (this) {
      //只能调用一次
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
      //又跑到dispatcher.executed(),黑人问号脸？
      client.dispatcher().executed(this);
      //重点，这里开始执行请求了并直接返回结果
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      //最后跑到dispatcher调用了finished()
      client.dispatcher().finished(this);
    }
  }
```

直接总结：

> 1.execute()只能被调用一次。

> 2.client.dispatcher().executed(this);这句干了什么？

```java
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

并没有很多动作，仅仅将这个Call加入到队列runningSyncCalls中。后面我们再来说说这个队列的事。

> 3.Response result = getResponseWithInterceptorChain();这里才是真正干活了。

getResponseWithInterceptorChain()源码：

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

上面代码先是添加了各种拦截器。先加一些从okHttpClient外部配置的一些拦截器，如果你用过OkHttpClient+Retrofit2，你很容易想到我们构建OkHttpClient的Builder的时候是会加一些我们自定义的拦截器的(这些我们自定义的拦截器是最先上车的)，后面又加一些OkHttp自带的拦截器：RetryAndFollowUpInterceptor...(我们先忽略掉这些拦截器，这正是这个库设计的精髓，这是后面我们要重点分析的东西，这一回合我们只分析请求过程)。这些裹上各种拦截器的请求作为一个最终的请求去服务器端申请数据了。

> 4.client.dispatcher().finished(this);

这里finished调用的是Dispatcher类的finish方法。同步请求里面没有过多操作，只是移除请求结束后将该请求从runningSyncCalls队列中移除。

```java
/** Used by {@code AsyncCall#run} to signal completion. */
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }

  /** Used by {@code Call#execute} to signal completion. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
       //将该请求从runningSyncCalls队列中移除
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
       //同步请求结束不会执行 promoteCalls()
      if (promoteCalls) promoteCalls();
       //返回同步请求队列和异步请求队列里面的请求总数
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }
	//下面这个if判断基本不会执行。因为源码中idleCallback一直为空
    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }
public synchronized int runningCallsCount() {
    return runningAsyncCalls.size() + runningSyncCalls.size();
  }
```

至此，同步请求的过程我们大概明白了。
```mermaid
graph LR;
    OkHttpClient-->|newCall| RealCall;
    RealCall-->|newRealCall| ;
    B-->D;
    C-->D;
```

