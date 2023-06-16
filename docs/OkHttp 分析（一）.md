# 1. OkHttp 调度器 Dispatcher

OkHttp 的请求主要在 RealCall 里面，同步请求方法 execute()，异步请求方法 enqueue()，看异步方法：

```kotlin
override fun enqueue(responseCallback: Callback) {
    synchronized(this) {
        check(!executed) { "Already Executed" }
        executed = true
    }
    callStart()
    client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```

可以看到，OkHttp 把请求 Runnable 交给了调度器 **Dispatcher** 执行。
Dispatcher 有 3 个 **ArrayDeque** 队列用于请求顺序，分别是 :

- **readyAsyncCalls** 用于异步请求，用于存储准备异步调用的请求
- **runningAsyncCalls** 用于异步其请求，用于存储正在执行的异步请求
- **runningSyncCalls** 用于同步请求，用于存储正在执行的同步请求

```kotlin
internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
        readyAsyncCalls.add(call)

        // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
        // the same host.
        if (!call.call.forWebSocket) {
            val existingCall = findExistingCallWithHost(call.host)
            if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
        }
    }
    promoteAndExecute()
}
```

在执行异步请求时，先将请求 Running 添加到 **readyAsyncCalls** 中，在 OkHttpClient 中调用 newCall 时，forWebSocket 为 false，**接下来会在当前的异步队列中查找是否存在与当前 call 的 host 相等的请求，如果有则返回 并且更新 callsPerHost**，callsPerHost 是 AtomicInterget 类型。（即** 地址相同的会共享socket，callPerHost代表当前线程数 **）

```kotlin
private fun findExistingCallWithHost(host: String): AsyncCall? {
    for (existingCall in runningAsyncCalls) {
        if (existingCall.host == host) return existingCall
    }
    for (existingCall in readyAsyncCalls) {
        if (existingCall.host == host) return existingCall
    }
    return null
}
```

```kotlin
fun reuseCallsPerHostFrom(other: AsyncCall) {
    this.callsPerHost = other.callsPerHost
}
```

接下来调用 promoteAndExecute 方法：

```kotlin
private fun promoteAndExecute(): Boolean {
    this.assertThreadDoesntHoldLock()

    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
        val i = readyAsyncCalls.iterator()
        while (i.hasNext()) {
            val asyncCall = i.next()

            if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
            if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.

            i.remove()
            asyncCall.callsPerHost.incrementAndGet()
            executableCalls.add(asyncCall)
            runningAsyncCalls.add(asyncCall)
        }
        isRunning = runningCallsCount() > 0
    }

    for (i in 0 until executableCalls.size) {
        val asyncCall = executableCalls[i]
        asyncCall.executeOn(executorService)
    }

    return isRunning
}
```

- 首先遍历 **readyAsyncCalls 取出请求，注意这个过程是加锁的。**
- 接下来判断如果正在执行的请求队列 runningAsyncCalls 大小超过 maxRequests（默认64），则跳出循环。如果上面说过的 callsPerHost 计数器大于 maxRequestsPerHost（默认值5）则跳出本次循环。
- 然后把当前请求在 readyAsyncCalls 中移除，线程数计数器 callsPerHost 加一，接着把当前请求添加到临时列表 executableCalls 中。把当前请求添加到正在执行队列 runningAsyncCalls 中。
- isRunning 的判断就是判断正在请求的队列大小是否大于 0 
- 最后遍历临时列表，并执行 RealCall 的 executeOn 方法。传入了 executorService 对象。



executorService 是 ExecutorService，它是一个线程池。适合执行多个时间短的任务。

```kotlin
fun executeOn(executorService: ExecutorService) {
    client.dispatcher.assertThreadDoesntHoldLock()

    var success = false
    try {
        executorService.execute(this)
        success = true
    } catch (e: RejectedExecutionException) {
        val ioException = InterruptedIOException("executor rejected")
        ioException.initCause(e)
        noMoreExchanges(ioException)
        responseCallback.onFailure(this@RealCall, ioException)
    } finally {
        if (!success) {
            client.dispatcher.finished(this) // This call is no longer running!
        }
    }
}
```

executeOn 方法主要就是交给线程池去执行请求。注意到如果没成功，会执行 Dispatcher 的 finish 方法：

```kotlin
internal fun finished(call: AsyncCall) {
    call.callsPerHost.decrementAndGet()
    finished(runningAsyncCalls, call)
}

private fun <T> finished(calls: Deque<T>, call: T) {
    val idleCallback: Runnable?
        synchronized(this) {
            if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
            idleCallback = this.idleCallback
        }

    val isRunning = promoteAndExecute()

    if (!isRunning && idleCallback != null) {
        idleCallback.run()
    }
}
```

finish 会先把线程计数器**减 1**，然后把当前请求在 **runningAsyncCalls** 队列中移除。

如果执行没问题的话，会执行 AsyncCall 的 run 方法：

```kotlin
override fun run() {
    threadName("OkHttp ${redactedUrl()}") {
        var signalledCallback = false
        timeout.enter()
        try {
            val response = getResponseWithInterceptorChain()
            signalledCallback = true
            responseCallback.onResponse(this@RealCall, response)
        } catch (e: IOException) {
            if (signalledCallback) {
                // Do not signal the callback twice!
                Platform.get().log("Callback failure for ${toLoggableString()}", Platform.INFO, e)
            } else {
                responseCallback.onFailure(this@RealCall, e)
            }
        } catch (t: Throwable) {
            cancel()
            if (!signalledCallback) {
                val canceledException = IOException("canceled due to $t")
                canceledException.addSuppressed(t)
                responseCallback.onFailure(this@RealCall, canceledException)
            }
            throw t
        } finally {
            client.dispatcher.finished(this)
        }
    }
}
```

可以看到请求最后也会执行 finished 方法。

**总结：**

1. **执行开始时会把请求加到准备执行的队列。**
2. **然后进行共享相同的 host 的请求。**
3. **然后遍历列表，取出请求，判断如果正在执行请求的队列大于64，则跳出循环，如果当前请求线程超过5个，则跳出本次循环。**
4. **然后将请求冲准备队列中移除，线程计数器加 1 ，并把请求添加到正在执行队列中。**
5. **最后通过线程池执行请求。请求成功和失败都会将请求从正在运行队列中移除，线程计数器减 1 。**

# 2. RealInterceptorChain 流程

```kotlin
 internal fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)

    val chain = RealInterceptorChain(
        call = this,
        interceptors = interceptors,
        index = 0,
        exchange = null,
        request = originalRequest,
        connectTimeoutMillis = client.connectTimeoutMillis,
        readTimeoutMillis = client.readTimeoutMillis,
        writeTimeoutMillis = client.writeTimeoutMillis
    )

    //...
    val response = chain.proceed(originalRequest)
    //...
  }
```

OkHttp 的拦截器执行顺序是：** RealInterceptorChain -> 自定义拦截器 -> RetryAndFollowUpInterceptor**
** -> BridgeInterceptor -> CacheInterceptor -> ConnectInterceptor -> CallServerInterceptor。**

RealInterceptorChain 的 proceed 方法里面就是进行一些判断，并执行下一个拦截器的 intercept 方法，注意第一次执行时 exchange 为 null。该拦截器的作用是将所有拦截器串联起来。
**
**
# 3. RetryAndFollowUpInterceptor
RetryAndFollowUpInterceptor 是负责失败重试和重定向的拦截器。**它是通过死循环来实现重连机制。**

核心代码：
```kotlin
while (true) {
    //...
    try {
        //得到最终的网络请求结果，如果没发生异常，则不需要重试
        response = realChain.proceed(request)
        //...
    } catch (e: RouteException) {
        //判断出现通过单个路由连接时出现问题的异常是否可以重连
        continue
    } catch (e: IOException) {
        //如果出现IO异常，判断能不能恢复
        continue 
    }
    //...
}
```

**详细分析：**

```kotlin
while (true) {
    call.enterNetworkInterceptorExchange(request, newExchangeFinder)
    try{
        //...
    }finally{
        call.exitNetworkInterceptorExchange(closeActiveExchange)
    }
}
```

在 while 循环一开始，会调用 RealCall 的 enterNetworkInterceptorExchange 方法创建 ExchangeFinder。

```kotlin
fun enterNetworkInterceptorExchange(request: Request, newExchangeFinder: Boolean) {
    check(interceptorScopedExchange == null)
    check(exchange == null) {
      "cannot make a new request because the previous response is still open: " +
          "please call response.close()"
    }
    if (newExchangeFinder) {
      this.exchangeFinder = ExchangeFinder(
          connectionPool,
          createAddress(request.url),
          this,
          eventListener
      )
    }
  }
```

参数 newExchangeFinder 表示如果不重试则为 true，并且可以执行新的路由。

在 finally 里面调用 exitNetworkInterceptorExchange 方法清空一些对象。exchange 和 interceptorScopedExchange 都在 ConnectInterceptor 中赋值。
> 该值与[exchange]相同，但仅限于网络拦截器的执行。 [exchange]字段的流结束时分配为null，这可以在网络拦截器返回之前或之后。

```kotlin
internal fun exitNetworkInterceptorExchange(closeExchange: Boolean) {
    check(!noMoreExchanges) { "released" }

    if (closeExchange) {
        exchange?.detachWithViolence()
        check(exchange == null)
    }

    interceptorScopedExchange = null
}
```

在 try 中，**通过拦截器的返回值拿到最终的网络请求结果，**之所以 RetryAndFollowUpInterceptor 要第一个执行，是因为如果在其他位置就拿不到最终的结果，拦截器的请求返回值是一层层传递的。在这个过程中，如果发生 catch , 就尝试去恢复。

```kotlin
try {
    response = realChain.proceed(request)
    newExchangeFinder = true
}
```

在 catch 中，首先判断是不是发生了 RouteException。RouteException 是会在获取一个 RealConnection 与 Socket 连接是抛出。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/450005/1583749543605-977bd747-6863-4513-b222-0ae8d33f2f41.png#align=left&display=inline&height=304&originHeight=294&originWidth=676&size=283816&status=done&style=none&width=698)

如果抛出了异常，会通过 **recover** 方法判断是否可以恢复：

```kotlin
private fun recover(
    e: IOException,
    call: RealCall,
    userRequest: Request,
    requestSendStarted: Boolean
): Boolean {
    //是否在 OkHttpClient 中配置了是否支持重连机制
    if (!client.retryOnConnectionFailure) return false

    // 如果连接已经发送，无法再次发送请求正文。
    if (requestSendStarted && requestIsOneShot(e, userRequest)) return false

    // 检测该异常是否是致命的。
    if (!isRecoverable(e, requestSendStarted)) return false

    // 是否有更多的路线
    if (!call.retryAfterFailure()) return false

    // For failure recovery, use the same route selector with a new connection.
    return true
  }
```
### 
### isRecoverable方法检测异常
不可恢复的几类异常：

- ProtocolException：协议异常
- SSLHandshakeException：ssl握手异常
- SSLPeerUnverifiedException：ssl证书校验异常

关于连接超时的异常处理：
  对于 SocketTimeoutException 的异常，表示连接超时异常，这个异常是可以进行重连的，即OkHttp内部对连接超时异常会进行自动重试处理。
```kotlin
if (e is InterruptedIOException) {
    return e is SocketTimeoutException && !requestSendStarted
}
```

同样地，在第二个 catch 中，判断是否发送了 IOException，然后通过 recover 判断是否可以恢复。

**如果可以请求可以恢复，会用 continue 跳出本次循环，在下一次循环中重新执行拦截器请求一遍结果。**

如果请求没发生异常，则可以进入下一步：

```kotlin
if (priorResponse != null) {
    response = response.newBuilder()
        .priorResponse(priorResponse.newBuilder()
                .body(null)
                .build())
        .build()
}
```

1. 如果 priorResponse 不为空，则把它保存到 response 中。

```kotlin
val exchange = call.interceptorScopedExchange
// 判断返回结果response，是否需要继续完善请求，例如证书验证等等
val followUp = followUpRequest(response, exchange)
```

2. 接下来拿到 exchange，类型是 Exchange 和 followUp，类型是 Request。

     来到这里，其实请求已经成功了，但是服务器返回的状态码不一定是 200。所以 **followUpRequest** 方法的逻辑是根据返回的状态码判断是否需要重定向，如果状态码是 300，301，302，303，307，308这几重定向相关的状态码的话，会调用 **buildRedirectRequest** 方法组装 Request 重定向。

```kotlin
private fun buildRedirectRequest(userResponse: Response, method: String): Request? {
    // Does the client allow redirects?
    if (!client.followRedirects) return null
    val location = userResponse.header("Location") ?: return null
    // Don't follow redirects to unsupported protocols.
    val url = userResponse.request.url.resolve(location) ?: return null
    //...
    return requestBuilder.url(url).build()
}
```

- 如果在 OkHttpClient 中设置了不允许重定向，则返回 null。
- 获取响应头 Location 值，这就是要重定向的地址。

3. 如果需要重定向的话，followUp 不为空，则返回，否则执行下一步判断重定向次数

```kotlin
if (++followUpCount > MAX_FOLLOW_UPS) {
    throw ProtocolException("Too many follow-up requests: $followUpCount")
}
```

重定向次数最大是 20 次，超过这次数就不再请求了。


# 4. BridgeInterceptor

该拦截器是第二个拦截器，主要作用有：

- 在请求前对 Header 进行处理，如果用户有设置就直接使用，如果没有设置就使用默认的；
- 通过 Chain 调用下一个拦截器进行网络请求；
- 对与返回的结果，进行 Gzip 解压缩， Header 以及 cookie 的处理；

## 1. 用户 Request 到网络 Request 的转换
用户发起一个请求往往只需要设置一个 url 和一些 RequestBody 即可，但是真正的网络请求没有这么简单，BridgeInterceptor 拦截器在原来的 request 基础上添加了很多请求头：

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
    val userRequest = chain.request()
    val requestBuilder = userRequest.newBuilder()

    val body = userRequest.body
    if (body != null) {
        val contentType = body.contentType()
        if (contentType != null) {
            requestBuilder.header("Content-Type", contentType.toString())
        }
    }
    //...
    val networkResponse = chain.proceed(requestBuilder.build())
    //...
    val responseBuilder = networkResponse.newBuilder()
        .request(userRequest)
    //...
    return responseBuilder.build()
}
```

### 1.1 Content-Type
首部字段 Content-Type 说明了**实体主体内对象的媒体类型**。和首部字段 Accept 一样，字段值用 type/subtype 形式赋值。参数 charset 使用 iso-8859-1 或 euc-jp 等字符集进行赋值。

### 1.2 Content-Length
首部字段 Content-Length 表明了实体主体部分的大小（单位是字节）。对实体主体进行内容编码传输时，不能再使用 Content-Length 首部字段。

### 1.3 Transfer-Encoding
首部字段 Transfer-Encoding 规定了传输报文主体时采用的编码方式。HTTP/1.1 的传输编码方式仅对分块传输编码有效。值为 chunked 表示请求体的内容大小是未知的。因此 Transfer-Encoding 与 Content-Length 两个首部不能共存。

```kotlin
val contentLength = body.contentLength()
if (contentLength != -1L) {
    requestBuilder.header("Content-Length", contentLength.toString())
    requestBuilder.removeHeader("Transfer-Encoding")
} else {
    requestBuilder.header("Transfer-Encoding", "chunked")
    requestBuilder.removeHeader("Content-Length")
}
```

### 1.4 Host  

- 首部字段 Host 会告知服务器，请求的资源所处的互联网**主机名和端口号**。**Host 首部字段在 HTTP/1.1 规范内是唯一一个必须被包含在请求内的首部字段。**
- 首部字段 Host 和以单台服务器分配多个域名的虚拟主机的工作机制有很密切的关联，这是首部字段 Host 必须存在的意义。
- 请求被发送至服务器时，请求中的主机名会用 IP 地址直接替换解决。但如果这时，**相同的 IP 地址下部署运行着多个域名，那么服务器就会无法理解究竟是哪个域名对应的请求。因此，就需要使用首部字段 Host 来明确指出请求的主机名。**若服务器未设定主机名，那直接发送一个空值即可。

### 1.5 Connection  
默认就是 “Keep-Alive”，就是一个 TCP 连接之后不会关闭，保持连接状态。

### 1.6 Accept-Encoding 
Accept-Encoding 首部字段用来告知服务器用户代理**支持的内容编码及内容编码的优先级顺序**。可一次性指定多种内容编码。

```kotlin
var transparentGzip = false
if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
    transparentGzip = true
    requestBuilder.header("Accept-Encoding", "gzip")
}
```


### 1.7 Cookie 
首部字段 Cookie 会告知服务器，当客户端想获得 HTTP 状态管理支持时，就会在请求中包含从服务器接收到的Cookie。接收到多个 Cookie 时，同样可以以多个 Cookie 形式发送。

### 1.8 User-Agent 
这个值根据 OkHttp 的版本不一样而不一样，**表示客户端的信息**。

通过上面这些操作，就对一个普通的 Request 添加了很多头信息，让其成为可以发送网络请求的 Request 。

## 2. 将这个 Request 进行网络请求得到响应 Response
```kotlin
val networkResponse = chain.proceed(requestBuilder.build())
```


## 3. 将网络请求回来的响应 Response 转化为用户可用的 Response

### 3.1 获取并保存服务器返回的 Cookie

```kotlin
cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

fun CookieJar.receiveHeaders(url: HttpUrl, headers: Headers) {
  if (this === CookieJar.NO_COOKIES) return

  val cookies = Cookie.parseAll(url, headers)
  if (cookies.isEmpty()) return

  saveFromResponse(url, cookies)
}
```


### 3.2 将网络请求返回的响应进行转化

```kotlin
if (transparentGzip &&
        "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
        networkResponse.promisesBody()) {
    val responseBody = networkResponse.body
    if (responseBody != null) {
        val gzipSource = GzipSource(responseBody.source())
        val strippedHeaders = networkResponse.headers.newBuilder()
            .removeAll("Content-Encoding")
            .removeAll("Content-Length")
            .build()
        responseBuilder.headers(strippedHeaders)
        val contentType = networkResponse.header("Content-Type")
        responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
    }
}
```

当 transparentGzip 为 true ，表示请求设置的 Accept-Encoding 是支持 gzip 压缩的，意思就是**告知服务器客户端是支持 gzip 压缩的**，然后再判断服务器的响应头 Content-Encoding 是否也是 gzip 压缩的，意思就是**响应体内容是否是经过 gzip 压缩的**，如果都成立的条件下，那么它会将 Resposonse.body().source() 的输入流 BufferedSource 转化为 **GzipSource** 类型，**这样的目的就是让调用者在使用 Response.body().string() 获取响应内容时就是以解压的方式进行读取流数据。**

最后返回 Response 对象。

继续查看 [OkHttp 分析（二）](https://www.yuque.com/espoir/urcfma/yeyf9z)
