# OkHttp 原理剖析

[TOC]

## 一、基本介绍

OkHttp是一个优秀的Http请求框架, 主要优点如下.

* 支持最新的HTTP/2和SPYD协议
* 使用连接池保证对网络请求的重用和数量控制
* 内有缓存机制, 不必手动管理缓存, 并减少不必要的网络请求和带宽
* 内有重试和重定向机制
* 使用构建者模式构建请求, 方便使用

## 二、基本使用

### 2.1 依赖配置

gradle依赖配置如下:

```gradle
implementation 'com.squareup.okhttp3:okhttp:3.14.1'
```

最新版本可以去[官网](https://square.github.io/okhttp/#download)查询下载

`3.14.1`版本最低Java版本为`Java 1.8`, 所以Android项目需要在gradle里面做如下配置:

```gradle
android.compileOptions {
		sourceCompatibility JavaVersion.VERSION_1_8
		targetCompatibility JavaVersion.VERSION_1_8
}
```

### 2.2 基本请求

以Get请求为例, 请求分为同步请求和异步请求

* 同步请求会在当前线程执行, 执行结果可以从返回值或者异常拿到(异常就是失败了呗...)
* 异步请求则将请求放入队列, 由线程池拿取执行, 执行结果由回调传递

```java
OkHttpClient    client  = new OkHttpClient();

Request         request = new Request.Builder().get().url("https://www.baidu.com").build();

// 这里是执行同步请求, 需要捕获IOException, 需要自己处理错误情况
// 执行时会阻塞当前线程
try {
  	Response response = client.newCall(request).execute();
} catch (IOException e) {
  	// 请求失败
  	e.printStackTrace();
}

// 这里是执行异步请求, 需要传入一个callback对象, 当任务执行完成后回调这个callback对象
client.newCall(request).enqueue(new Callback() {
 	  @Override
  	public void onFailure(Call call, IOException e) {
      // 请求失败
   	  Log.e(TAG, "onFailure");
 	  }

 	 @Override
  	public void onResponse(Call call, Response response) throws IOException {
    	// 传递请求成功的结果
    	Log.e(TAG, "onResponse: " + response.toString());
    	Log.e(TAG, "onResponse: " + response.body().string());
  	}
});
```

## 三、原理剖析

### 3.1 创建请求

创建同步请求方式如下:

```java
OkHttpClient    client  = new OkHttpClient();
Request         request = new Request.Builder().get().url("https://www.baidu.com").build();
Call            call    = client.newCall(request);
```

1. 首先创建`Request.Builder`, 典型的创建者模式, 通过调用`get`,`url`等方法设置请求参数

```java
 public static class Builder {
    @Nullable HttpUrl url;
    String method;
    Headers.Builder headers;
    @Nullable RequestBody body;
    ...
 }
```

如上所示, Builder内部主要保存了`HttpUrl`, `method`, `Headers.Builder`, `RequestBody`参数对象, 这些参数除了`method`之外, 基本是通过构建者模式创建的, 大同小异, 这里不再赘述

2. 创建请求对象`Call`

```java
Call call = client.newCall(request);
```

调用`OkHttpClient`的`newCall`方法, 具体实现如下:

```java
public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
}
```

出现了`RealCall`, 这个是什么东西? 继续追踪...

```java
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.transmitter = new Transmitter(client, call);
    return call;
}
```

原来返回的`Call`对象就是`RealCall`, `RealCall`实现了`Call`接口, 成员变量主要如下:

```java
final class RealCall implements Call {
  	// 全局的OkHttpClient对象
    final OkHttpClient client;
  	// 中文叫做发射器, 作为应用层和底层的交互中介,
  	// 执行如准备连接、取消、超时之类的真正操作
    private Transmitter transmitter;
  	// 初始请求参数对象
    final Request originalRequest;
    final boolean forWebSocket;
  	// 是否已经执行过, 每个RealCall只能执行一次, 
  	// 否则抛出IllegalStateException("Already Executed")异常
    private boolean executed;
}
```

### 3.2 执行同步请求

#### 3.2.1 执行同步请求前夕

接下来看看`Call`的同步执行请求, 先跟一次代码:

```java
Response response = call.execute();
```

```java
@Override public Response execute() throws IOException {
    synchronized (this) {
      	if (executed) throw new IllegalStateException("Already Executed");
      	executed = true;
    }
    transmitter.timeoutEnter();
    transmitter.callStart();
    try {
      	client.dispatcher().executed(this);
      	return getResponseWithInterceptorChain();
    } finally {
      	client.dispatcher().finished(this);
    }
}
```


* 这里整体展现了请求最初始的信息
    1. 首先, 判断下是否执行过, 执行过就直接抛出异常
    2. `ransmitter.timeoutEnter()`开始记录超时, 超时通过`AsyncTimeout`这个类实现, 内部维护一个链表和`WatchDog`的线程子类, 使用`AsyncTimeout.class`的`wait/notify`来阻塞/通知`WatchDog`线程实现超时监听
    3. `transmitter.callStart()`开始通知监听器组件网络请求开始了, 主要是通知`EventListener`
    4. `client.dispatcher().executed(this)`告知`OkHttpClient`有个同步网络请求正在执行, `OkHttpClient`会将这个请求放置到`Deque<RealCall> runningSyncCalls`队列中
    5. `getResponseWithInterceptorChain`真正执行网络请求的地方, 下一步再说
    6. `client.dispatcher().finished(this)`通知`OkHttpClient`本请求执行完毕, 可以将自己从队列中取出来, 并检查是否有异步请求可以执行

接下来进入到`getResponseWithInterceptorChain`方法, 查看下细节, 先贴代码

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
  	// 构建拦截器
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

  	// 生成拦截器调用链
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
        originalRequest, this, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    boolean calledNoMoreExchanges = false;
    try {
      // 开始执行拦截器调用链
      Response response = chain.proceed(originalRequest);
      if (transmitter.isCanceled()) {
        closeQuietly(response);
        throw new IOException("Canceled");
      }
      return response;
    } catch (IOException e) {
      calledNoMoreExchanges = true;
      throw transmitter.noMoreExchanges(e);
    } finally {
      if (!calledNoMoreExchanges) {
        transmitter.noMoreExchanges(null);
      }
    }
}
```

#### 3.2.2 构建拦截器和调用链

首先开始那几行是构建拦截器数组, 主要使用到了策略模式(我觉得设计这个拦截器的大佬应该对JavaEE比较熟悉). 过程首先是给OkHttpClient配置的拦截器, 然后依次添加`RetryAndFollowUpInterceptor`, `BridgeInterceptor`, `CacheInterceptor`, `ConnectInterceptor`, `CallServerInterceptor`, 这些拦截器主要作用如下:

* `RetryAndFollowUpInterceptor`: 跟踪重试拦截器, 主要用于请求失败后重试、跟随重定向(302)
* `BridgeInterceptor`: 桥接拦截器, 主要功能是将Request参数转换为Http参数, 将Http结果封装为Response
* `CacheInterceptor`: 缓存拦截器, 查看请求是否有缓存, 如果包含则封装缓存为Response, 否则通过网络请求获取数据, 有时需要根据HTTP状态码304判断缓存是否有效
* `ConnectInterceptor`: 网络连接拦截器, 只是与服务器创建连接, 主要作用是获取`RealConnection`, 根据Http的Host, 首先会从`RealConnectionPool`获取已有的连接, 如果未找到, 则新建一个连接, 并封装为`RealConnection`放入`RealConnectionPool`, `RealConnection`底层基于Socket实现
* `CallServerInterceptor`: 服务器数据交互拦截器, 这里是真真正正的和服务器进行打交道了, 包括向服务器写数据, 从服务器读数据. `ExchangeCodec`是一个接口, 定义了读取、写入之类的操作, 下面有两个子类`Http1ExchangeCodec`和`Http2ExchangeCodec`, 分别针对于HTTP/1.1和HTTP/2. 

构建拦截器数组集合的目的就是为了能够链式调用拦截器. 这里主要用到了责任链模式. 

```java
new RealInterceptorChain(interceptors, transmitter, null, 0, originalRequest, this, ...);
```

在构造调用链时, 第一个参数为拦截器数组集合, 第四个为**这个`RealInterceptorChain`对应的`Interceptor`**下标, 此时, 任何请求的操作还没开始.

#### 3.2.3 调用拦截器调用链

终于到了调用拦截器链的地方了, 入口代码如下:

```java
Response response = chain.proceed(originalRequest);
```

就这么一句就把所有的拦截器都调用了? 进去看看:

```java
@Override public Response proceed(Request request) throws IOException {
    return proceed(request, transmitter, exchange);
}
```

`proceed`是接口`Interceptor.Chain`定义的函数, 由`RealInterceptorChain`实现的, 真正的实现调用了如下方法: 

```java
public Response proceed(Request request, Transmitter transmitter, Exchange exchange) 
  	throws IOException {
		...

    // 创建下一个调用链对象
    RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
        index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
  
  	// 获取本RealInterceptorChain对象对应Interceptor
    Interceptor interceptor = interceptors.get(index);
  	// 调用本RealInterceptorChain对象对应Interceptor
    Response response = interceptor.intercept(next);
  
  	...
    return response;
  }
```

`proceed(Request, Transmitter, Exchange) `主要做了三件事:

1. 检查状态、参数是否合法(为了版本省略了代码)
2. 创建下一个调用链对象
3. 调用与本`RealInterceptorChain`对象对应`Interceptor`的`intercept`方法(这里可能有点绕)

来看看调用栈:

![请求调用栈](Untitled.assets/blog_okhttp_interceptorchain_stacktrace.png)

调用栈显示, 这些拦截器一个一个的向上堆叠, 后执行的拦截器执行完后会回到先执行的拦截器里, 如果后执行的拦截器结果是失败的, 先执行的拦截器可以重新进行请求, 再来一次类似的拦截器调用, 其实`RetryAndFollowUpInterceptor`就是这么玩的.

### 3.3 执行异步请求

#### 3.3.1 执行同步请求前夕

异步请求执行方式如下:

```java
call.enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        Log.e(TAG, "onFailure");
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        Log.e(TAG, "onResponse: " + response.toString());
        Log.e(TAG, "onResponse: " + response.body().string());
    }
});
```

`enqueue`? 加入队列了吗? 是的, 就是加入队列了, 跟进一步看看:

```java
public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.callStart();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

做的很常规, 检查请求是否执行过; 调用`EventListener`的`callStart`方法; 请求入队列. 但最后一句是关键: 先将`Callback`对象封装为`AsyncCall`, `AsyncCall`是`RealCall`的内部类(不是静态内部类), 并且实现了`NamedRunnable`接口, 然后将`AsyncCall`加入到`OkHttpClient`的`Dispatcher`里面.

#### 3.3.2 请求入队

继续跟进`enqueue`方法:

```java
void enqueue(AsyncCall call) {
    synchronized (this) {
        readyAsyncCalls.add(call);
        ...
    }
    promoteAndExecute();
}
```

上述方法主要操作如下:

* 将`AsyncCall`加入到队列`readyAsyncCalls`中
* `promoteAndExecute`检查请求并迭代执行, 主要是将队列`readyAsyncCalls`的请求拿出来放到`runningAsyncCalls`中

再来跟踪`promoteAndExecute`方法:

```java
private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        asyncCall.callsPerHost().incrementAndGet();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }

    return isRunning;
}
```

有两个成员变量: `maxRequests`和`maxRequestsPerHost`. `maxRequests`标识最多有多少个请求可以同时进行, 不管同步异步, 默认是`64`;`maxRequestsPerHost`标识每个Host最多有多少个请求可以同时进行, 不管同步异步, 默认是`5`

上述代码前半部分主要是摘取出可以执行的异步请求, 并将其加入到`runningAsyncCalls`队列中. 后半部分就是将摘取出的异步请求放入线程池异步执行.

#### 3.3.3 请求异步执行

线程池异步执行, 应该要有`run`方法, 找找`AsyncCall`的`run`方法: 

```java
public abstract class NamedRunnable implements Runnable {
    protected final String name;

    public NamedRunnable(String format, Object... args) {
        this.name = Util.format(format, args);
    }

    @Override public final void run() {
        String oldName = Thread.currentThread().getName();
        Thread.currentThread().setName(name);
        try {
            execute();
        } finally {
            Thread.currentThread().setName(oldName);
        }
    }

    protected abstract void execute();
}
```

`run`方法用于设置线程名称了, 代替的是`execute`方法, 继续找:


```java
final class AsyncCall extends NamedRunnable {
    ...

    @Override
    protected void execute() {
        boolean signalledCallback = false;
        transmitter.timeoutEnter();
        try {
            Response response = getResponseWithInterceptorChain();
            signalledCallback = true;
            responseCallback.onResponse(RealCall.this, response);
        } catch (IOException e) {
            if (signalledCallback) {
                // Do not signal the callback twice!
                Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
            } else {
                responseCallback.onFailure(RealCall.this, e);
            }
        } finally {
            client.dispatcher().finished(this);
        }
    }
}
```

是不是有点眼熟, 是不是和`3.2.1`节差不多, 再贴一遍代码:

```java
@Override public Response execute() throws IOException {
    synchronized (this) {
      	if (executed) throw new IllegalStateException("Already Executed");
      	executed = true;
    }
    transmitter.timeoutEnter();
    transmitter.callStart();
    try {
      	client.dispatcher().executed(this);
      	return getResponseWithInterceptorChain();
    } finally {
      	client.dispatcher().finished(this);
    }
}
```

到这里剩下的流程就和同步请求的流程一致了, 就要走`getResponseWithInterceptorChain`这一套了, 这里不再赘述, 请回看`3.2.1`节

## 致谢