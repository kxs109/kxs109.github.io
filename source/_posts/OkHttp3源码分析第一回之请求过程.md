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
- execute方法
- enqueue方法 

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

当然它使用的的参数是Builder里面默认设置的参数：

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

当然了，我们也可以像开头代码示例一样不采用默认的Builder参数，我们自由设置自己的Builder参数，然后再build出一个OkHttpClient实例：

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

这是一个重写了Call接口的内部接口Factory的call方法，然后调用的是RealCall的newRealCall方法。

Android的网络请求框架有很多选择，不过Square公司出品的OkHttp无疑是最杰出的。其源码融合的设计思想非常值得深入研究。