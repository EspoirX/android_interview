# 1. CacheInterceptor
  该拦截器主要处理缓存，OkHttp 通过 OkHttpClient 配置缓存：

```java
File cacheFile = new File(getExternalCacheDir().toString(),"cache");
int cacheSize = 10 * 1024 * 1024;
Cache cache = new Cache(cacheFile,cacheSize);
OkHttpClient client = new OkHttpClient.Builder()
    .cache(cache)
    .build();
```

 缓存对象为 Cache。
```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
    val cacheCandidate = cache?.get(chain.request())
    //...
}
```

首先从 Cache 中通过 get 方法获取本地缓存 **cacheCandidate（Response）**

```kotlin
internal fun get(request: Request): Response? {
    val key = key(request.url)
    val snapshot: DiskLruCache.Snapshot = try {
        cache[key] ?: return null
    } catch (_: IOException) {
        return null // Give up because the cache cannot be read.
    }

    val entry: Entry = try {
        Entry(snapshot.getSource(ENTRY_METADATA))
    } catch (_: IOException) {
        snapshot.closeQuietly()
        return null
    }

    val response = entry.response(snapshot)
    if (!entry.matches(request, response)) {
        response.body?.closeQuietly()
        return null
    }

    return response
}
```

get 方法中 cache 是一个 **DiskLruCache** 对象，key 是编码过后的 url。

-  首先通过 key 来尝试获取一个 Snapshot，在 DiskLruCache 中，如果 Snapshot 存在，则移动到 Lru 队列的头部来，通过 Snapshot 可以得到一个输入流 InputStream。
- 通过 Snapshot#getSource 获取队列第一个未缓存的流，并创建 Entry。ENTRY_METADATA 的值为 0。在 Entry 的构造方法中，会从 source 中获取对应的信息并给相关的类赋值。
- 通过 Entry 的 response 方法从缓存信息中创建并返回一个 **Response**
- 通过 Entry 的 matches 方法判断是否是同一个请求，如果是的话返回 Response，否则返回 null

接下来通过时间，原始 Request，和 cacheCandidate 构造出  **CacheStrategy，并调用 compute 方法**。

```kotlin
val now = System.currentTimeMillis()

val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
val networkRequest = strategy.networkRequest
val cacheResponse = strategy.cacheResponse
```

## CacheStrategy
CacheStrategy 有两个重要的变量 networkRequest 和 cacheResponse。如果 **networkRequest 为 null 就不走网络取数据，cacheResponse 为 null 则不用缓存。** 

```kotlin
fun compute(): CacheStrategy {
    val candidate = computeCandidate()

    // We're forbidden from using the network and the cache is insufficient.
    if (candidate.networkRequest != null && request.cacheControl.onlyIfCached) {
        return CacheStrategy(null, null)
    }

    return candidate
}
```

核心是 computeCandidate 方法：

```kotlin

if (cacheResponse == null) {
    return CacheStrategy(request, null)
}
 
if (request.isHttps && cacheResponse.handshake == null) {
    return CacheStrategy(request, null)
}
 
if (!isCacheable(cacheResponse, request)) {
    return CacheStrategy(request, null)
}

val requestCaching = request.cacheControl
if (requestCaching.noCache || hasConditions(request)) {
    return CacheStrategy(request, null)
}
```

1. 先有几个判断，如果 cacheResponse 为空，则直接使用网络数据，cacheResponse 即为上面的 cacheCandidate。
2. 如果请求是 https 且缓存没有 TLS 握手，则直接使用网络数据。
3. 如果响应不该被缓存，则是用网络数据，isCacheable 方法判断响应该不该缓存，通过返回码判断。
4. 如果缓存策略指定了不使用缓存，以及请求头部带有条件缓存的相关首部字段，使用网络数据。

唯一使用缓存的地方：


```kotlin
val responseCaching = cacheResponse.cacheControl

val ageMillis = cacheResponseAge() //响应的当前时间
var freshMillis = computeFreshnessLifetime() //最新的响应时间

if (requestCaching.maxAgeSeconds != -1) {
    freshMillis = minOf(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds.toLong()))
}

var minFreshMillis: Long = 0
if (requestCaching.minFreshSeconds != -1) {
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds.toLong())
}

var maxStaleMillis: Long = 0
if (!responseCaching.mustRevalidate && requestCaching.maxStaleSeconds != -1) {
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds.toLong())
}

if (!responseCaching.noCache && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    val builder = cacheResponse.newBuilder()
    if (ageMillis + minFreshMillis >= freshMillis) {
        builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"")
    }
    val oneDayMillis = 24 * 60 * 60 * 1000L
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
        builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"")
    }
    return CacheStrategy(null, builder.build())
}
```

如果满足条件 **!responseCaching.noCache && ageMillis + minFreshMillis < freshMillis + maxStaleMillis**
则使用缓存。

**总结：**

- request 的 header 有 only-if-cached：啥缓存都不用;
- cacheResponse 为 null，缓存中为空，当然不使用缓存;
- request.isHttps() && cacheResponse.handshake() == null，为 https 且缓存丢失了TLS握手：不用缓存;
- reques t或者 response 的 header 有 no-store：不用缓存;
- response 除了 200 一些的 status code 以外：不用缓存;
- request 的 CacheControl 指定了不使用缓存：不用缓存;
- request 含有条件请求相关的首部，If-Modified-Since 或者 If-None-Match，不用缓存;
- 满足这个条件 ageMillis + minFreshMillis < freshMillis + maxStaleMillis 的 request 和 response：用缓存;


回到 CacheInterceptor 中，在获取到 networkRequest 和 cacheResponse 后，首先判断，如果两个都为空，则返回一个空的 Response：


```kotlin
if (networkRequest == null && cacheResponse == null) {
    return Response.Builder()
        .request(chain.request())
        .protocol(Protocol.HTTP_1_1)
        .code(HTTP_GATEWAY_TIMEOUT)
        .message("Unsatisfiable Request (only-if-cached)")
        .body(EMPTY_RESPONSE)
        .sentRequestAtMillis(-1L)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build()
}
```


如果使用缓存数据，则返回缓存的 Response:

```kotlin
if (networkRequest == null) {
    return cacheResponse!!.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .build()
}
```

否则通过网络请求获取网络 Response:

```kotlin
var networkResponse: Response? = null
try {
    networkResponse = chain.proceed(networkRequest)
} finally {
    // If we're crashing on I/O or otherwise, don't leak the cache body.
    if (networkResponse == null && cacheCandidate != null) {
        cacheCandidate.body?.closeQuietly()
    }
}
```

如果这时候缓存数据也不为空的话，则判断请求返回码是否是 304，如果是的话则更新缓存并返回：

```kotlin
if (cacheResponse != null) {
    if (networkResponse?.code == HTTP_NOT_MODIFIED) {
        val response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers, networkResponse.headers))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build()

        networkResponse.body!!.close()

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache!!.trackConditionalCacheHit()
        cache.update(cacheResponse, response)
        return response
    } else {
        cacheResponse.body?.closeQuietly()
    }
}
```

最后把缓存数据保存到 networkRequest 中，并判断如果该缓存，则 put 到缓存中，并返回，否则从缓存中移除。

```kotlin
val response = networkResponse!!.newBuilder()
    .cacheResponse(stripBody(cacheResponse))
    .networkResponse(stripBody(networkResponse))
    .build()

if (cache != null) {
    if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        val cacheRequest = cache.put(response)
        return cacheWritingResponse(cacheRequest, response)
    }

    if (HttpMethod.invalidatesCache(networkRequest.method)) {
        try {
            cache.remove(networkRequest)
        } catch (_: IOException) {
            // The cache cannot be written.
        }
    }
}
```

缓存的存取最终都是封装成类 Entry ，写缓存最终实现是在 Entry 的 writeTo 方法进行的，


# 2. ConnectInterceptor
该拦截器负责打开与目标服务器的连接，然后进入下一个拦截器。代码看试简单，其实很复杂。

```kotlin
object ConnectInterceptor : Interceptor {
  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.call.initExchange(chain)
    val connectedChain = realChain.copy(exchange = exchange)
    return connectedChain.proceed(realChain.request)
  }
}
```

首先是通过 RealCall 的 initExchange 方法获取到 Exchange 对象。

```kotlin
internal fun initExchange(chain: RealInterceptorChain): Exchange {
    synchronized(connectionPool) {
        check(!noMoreExchanges) { "released" }
        check(exchange == null)
    }

    val codec = exchangeFinder!!.find(client, chain)
    val result = Exchange(this, eventListener, exchangeFinder!!, codec)
    this.interceptorScopedExchange = result

    synchronized(connectionPool) {
        this.exchange = result
        this.exchangeRequestDone = false
        this.exchangeResponseDone = false
        return result
    }
}
```

经过分析，知道 exchangeFinder 对象是在 RetryAndFollowUpInterceptor 拦截器中创建的，下面看它的 find 方法：

```kotlin
fun find(
    client: OkHttpClient,
    chain: RealInterceptorChain
): ExchangeCodec {
    try {
        val resultConnection = findHealthyConnection(
            connectTimeout = chain.connectTimeoutMillis,
            readTimeout = chain.readTimeoutMillis,
            writeTimeout = chain.writeTimeoutMillis,
            pingIntervalMillis = client.pingIntervalMillis,
            connectionRetryEnabled = client.retryOnConnectionFailure,
            doExtensiveHealthChecks = chain.request.method != "GET"
        )
        //找到健康的连接，调用 newCodec 方法创建 ExchangeCodec
        return resultConnection.newCodec(client, chain)
    } catch (e: RouteException) {
        trackFailure(e.lastConnectException)
        throw e
    } catch (e: IOException) {
        trackFailure(e)
        throw RouteException(e)
    }
}
```

find 方法通过 findHealthyConnection 方法查找出健康的 RealConnection 对象，并且通过它的 newCodec 方法创建并返回 ExchangeCodec。

### 2.1 findHealthyConnection 流程

```kotlin
private fun findHealthyConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    doExtensiveHealthChecks: Boolean
): RealConnection {
    while (true) {
        val candidate = findConnection(
            connectTimeout = connectTimeout,
            readTimeout = readTimeout,
            writeTimeout = writeTimeout,
            pingIntervalMillis = pingIntervalMillis,
            connectionRetryEnabled = connectionRetryEnabled
        )

        // Confirm that the connection is good. If it isn't, take it out of the pool and start again.
        if (!candidate.isHealthy(doExtensiveHealthChecks)) {
            candidate.noNewExchanges()
            continue
        }

        return candidate
    }
}
```

- 通过 while 循环来寻找一个连接
- 通过 findConnection 方法查找连接，并且通过 isHealthy 方法判断该链接是否健康。
- 如果健康，则返回，否则进行一系列的销毁操作并进行下一次循环。

### 2.2 findConnection

```kotlin
var foundPooledConnection = false        //是否在连接池中找到  Connection
var result: RealConnection? = null       //找到的可用连接
var selectedRoute: Route? = null         //找到的路由
var releasedConnection: RealConnection?  //可以释放的连接
val toClose: Socket?                     //需要关闭的连接
```

findConnection 中几个临时变量的意思。

首先会给 releasedConnection 和 toClose 赋值：

```kotlin
releasedConnection = call.connection
toClose = if (call.connection != null &&
        (call.connection!!.noNewExchanges || !call.connection!!.supportsUrl(address.url))) {
    call.releaseConnectionNoEvents()
} else {
    null
}
```

然后会尝试复用当前在 RealCall 中的连接：


```kotlin
if (call.connection != null) { //尝试复用当前的 RealConnection
    result = call.connection
    releasedConnection = null
}
```

如果当前的 RealConnection 不能复用，则在连接池里面获取一个请求。

```kotlin
//如果当前的 RealConnection 不能复用，则在连接池里面获取一个请求
if (result == null) {
  
    refusedStreamCount = 0
    connectionShutdownCount = 0
    otherFailureCount = 0

    //尝试从池中获取连接。
    if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
        foundPooledConnection = true //修改标志位
        result = call.connection
    } else if (nextRouteToTry != null) {
        selectedRoute = nextRouteToTry
        nextRouteToTry = null
    }
}
```

这里看 **callAcquirePooledConnection** 方法：

```kotlin
fun callAcquirePooledConnection(
    address: Address,
    call: RealCall,
    routes: List<Route>?,
    requireMultiplexed: Boolean
): Boolean {
    this.assertThreadHoldsLock()

    for (connection in connections) {
        if (requireMultiplexed && !connection.isMultiplexed) continue
        //判断连接是否可以复用
        if (!connection.isEligible(address, routes)) continue
        call.acquireConnectionNoEvents(connection)
        return true
    }
    return false
}
```

核心是通过 **isEligible** 方法判断连接是否可以复用，如果可以复用，则通过 RealCall 的 acquireConnectionNoEvents 方法更新 RealCall 中当前的连接。

**isEligible 方法的判断逻辑：**

- 当前的连接的最大并发数不能达到上限，否则不能复用
- 两个连接的 address 的 Host 不相同，不能复用
- 1、2 步通过后，url 的 host 相同则可以复用
- 如果第 3 步中 url 的 host 不相同，可以通过合并连接实现复用
- 但首先这个连接需要是 HTTP/2
- 不能是代理
- IP 的 address 要相同
- 这个连接的服务器证书必须覆盖新的主机
- 证书将必须匹配主机
- 以上都不行，则这个连接就不能复用

**如果找到了可复用连接，则回调并直接返回。**
**
如果找不到可复用连接，会判断是否需要路由切换，如果需要，则遍历路由列表并根据条件选择一个，是一个阻塞操作：


```kotlin
var newRouteSelection = false
//给 routeSelector 和 routeSelection 赋值
if (selectedRoute == null && (routeSelection == null || !routeSelection!!.hasNext())) {
    var localRouteSelector = routeSelector
    if (localRouteSelector == null) {
        localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
        this.routeSelector = localRouteSelector
    }
    newRouteSelection = true
    routeSelection = localRouteSelector.next()
}
```

**选择完路由后，会再次尝试寻找可以复用的连接：**
**
```kotlin
if (newRouteSelection) {
    //现在我们有了一组IP地址，再次尝试从池中获取连接。 由于连接合并，这可能匹配。
    routes = routeSelection!!.routes
    if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
        foundPooledConnection = true
        result = call.connection
    }
}
```

**如果找到，则返回，如果还是找不到，则创建一个新的连接：**

```kotlin
if (!foundPooledConnection) {
    if (selectedRoute == null) {
        selectedRoute = routeSelection!!.next()
    }
    result = RealConnection(connectionPool, selectedRoute!!)
    connectingConnection = result
}
```

**对于新创建的连接，需要进行 connect 操作进行连接：**

```kotlin
result!!.connect(
    connectTimeout,
    readTimeout,
    writeTimeout,
    pingIntervalMillis,
    connectionRetryEnabled,
    call,
    eventListener
)
call.client.routeDatabase.connected(result!!.route())
```

#### 2.2.1 connect 连接流程

```kotlin
fun connect(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    call: Call,
    eventListener: EventListener
) {
    check(protocol == null) { "already connected" }

    var routeException: RouteException? = null
    val connectionSpecs = route.address.connectionSpecs
    val connectionSpecSelector = ConnectionSpecSelector(connectionSpecs)

    if (route.address.sslSocketFactory == null) {
        if (ConnectionSpec.CLEARTEXT !in connectionSpecs) {
            throw RouteException(UnknownServiceException(
                "CLEARTEXT communication not enabled for client"))
        }
        val host = route.address.url.host
        if (!Platform.get().isCleartextTrafficPermitted(host)) {
            throw RouteException(UnknownServiceException(
                "CLEARTEXT communication to $host not permitted by network security policy"))
        }
    } else {
        if (Protocol.H2_PRIOR_KNOWLEDGE in route.address.protocols) {
            throw RouteException(UnknownServiceException(
                "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"))
        }
    }
    //连接开始
    while (true) {
        try {
            //如果要求隧道模式，建立通道连接，通常不是这种
            if (route.requiresTunnel()) {
                connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener)
                if (rawSocket == null) {
                    // We were unable to connect the tunnel but properly closed down our resources.
                    break
                }
            } else {
                //一般都是这种逻辑，实际上就是 socket 连接
                connectSocket(connectTimeout, readTimeout, call, eventListener)
            }
            //https 的建立
            establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)
            eventListener.connectEnd(call, route.socketAddress, route.proxy, protocol)
            break
        } catch (e: IOException) {
            //...
        }
    }
    //...
    idleAtNs = System.nanoTime()
}
```

- 检查连接是否已经建立，若已经建立，则抛出异常，否则继续，连接的是否简历由 protocol 标示，它表示在整个连接建立，及可能的协商过程中选择所有要用到的协议。
- 用集合 **connnectionspecs** 构造 **ConnectionSpecSelector**。
- 如果请求是不安全的请求，会对请求执行一些额外的限制:
 

      (1). ConnectionSpec 集合必须包含 ConnectionSpec.CLEARTEXT。也就是说 OkHttp 用户可以通过 OkHttpClient 设置不包含 ConnectionSpec.CLEARTEXT 的 ConnectionSpec 集合来禁用所有的明文要求。
      (2). 平台本身的安全策略允向相应的主机发送明文请求。对于Android平台而言，这种安全策略主要由系统的组件 android.security.NetworkSecurityPolicy 执行。平台的这种安全策略不是每个 Android 版本都有             的。Android6.0之后存在这种控制。(okhttp/okhttp/src/main/java/okhttp3/internal/platform/AndroidPlatform.java 里面的isCleartextTrafficPermitted()方法)

- 根据请求判断是否需要建立隧道连接，如果建立隧道连接则调用 **connectTunnel**
- 如果不是隧道连接则调用 **connectSocket** 建立普通连接。
- 通过调用 **establishProtocol** 建立协议
- 如果是 HTTP/2，则设置相关属性。



#### 2.2.2 connectSocket

```kotlin
private fun connectSocket(
    connectTimeout: Int,
    readTimeout: Int,
    call: Call,
    eventListener: EventListener
) {
    val proxy = route.proxy
    val address = route.address
    //根据代理类型来选择 socket 类型，是代理还是直连
    val rawSocket = when (proxy.type()) {
        Proxy.Type.DIRECT, Proxy.Type.HTTP -> address.socketFactory.createSocket()!!
            else -> Socket(proxy)
    }
    this.rawSocket = rawSocket //给变量赋值
    //开始连接回调
    eventListener.connectStart(call, route.socketAddress, proxy)
    rawSocket.soTimeout = readTimeout //设置超时时间
    try {
        //连接 socket，之所以这样写是因为支持不同的平台
        //里面实际上是  socket.connect(address, connectTimeout);
        Platform.get().connectSocket(rawSocket, route.socketAddress, connectTimeout)
    } catch (e: ConnectException) {
        throw ConnectException("Failed to connect to ${route.socketAddress}").apply {
            initCause(e)
        }
    }
    // 得到输入／输出流
    try {
        source = rawSocket.source().buffer()
        sink = rawSocket.sink().buffer()
    } catch (npe: NullPointerException) {
        if (npe.message == NPE_THROW_WITH_NULL) {
            throw IOException(npe)
        }
    }
}
```

**有 3 种情况需要建立普通连接：**

- 无代理
- 明文的HTTP代理
- SOCKS代理

**普通连接的建立过程为建立 TCP 连接，建立 TCP 连接的过程为：**

- 创建 Socket，非 SOCKS 代理的情况下，通过 **SocketFactory** 创建；在 SOCKS 代理则传入 proxy 手动new 一个出来。
- 为 Socket 设置超时
- 完成特定于平台的连接建立
- 创建用于 I/O 的 source 和 sink

设置了 SOCKS 代理的情况下，仅有的特别之处在于，是通过传入 proxy 手动创建 Socket。  
route 的 socketAddress 包含目标 HTTP 服务器的域名。由此可见 SOCKS 协议的处理，主要是在 Java 标准库的 java.net.Socket 中处理，对于外界而言，就好像是HTTP服务器直接建立连接一样，因此连接时传入的地址都是HTTP 服务器的域名。  
而对于明文的 HTTP 代理的情况下，这里没有任何特殊处理。route 的 socketAddress 包含着代理服务器的 IP 地址。HTTP 代理自身会根据请求及相应的实际内容，建立与 HTTP 服务器的 TCP 连接，并转发数据。

#### 2.2.3 建立隧道逻辑 connectTunnel

```kotlin
private fun connectTunnel(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    call: Call,
    eventListener: EventListener
) {
    var tunnelRequest: Request = createTunnelRequest()
    val url = tunnelRequest.url
    for (i in 0 until MAX_TUNNEL_ATTEMPTS) {
        connectSocket(connectTimeout, readTimeout, call, eventListener)
        tunnelRequest = createTunnel(readTimeout, writeTimeout, tunnelRequest, url)
        ?: break // Tunnel successfully created.

        // The proxy decided to close the connection after an auth challenge. We need to create a new
        // connection, but this time with the auth credentials.
        rawSocket?.closeQuietly()
        rawSocket = null
        sink = null
        source = null
        eventListener.connectEnd(call, route.socketAddress, route.proxy, null)
    }
}
```

- 在隧道模式下，首先通过 createTunnelRequest 方法创建隧道请求
- 然后通过 connectSocket 建立 Socket 连接
- 然后发送请求建立隧道

总结：

- 在没有设置代理的情况下，直接与 HTTP 服务器建立 TCP 连接，然后进行 HTTP 请求/响应的交互。
- 设置了 SOCKS 代理的情况下，创建 Socket 时，为其传入 proxy，连接时还是以 HTTP服 务器为目标。在标准库的 Socket 中完成 SOCKS 协议相关的处理。此时基本上感知不到代理的存在。
- 设置了 HTTP 代理时的 HTTP 请求，与 HTTP 代理服务器建立 TCP 连接。HTTP 代理服务器解析 HTTP 请求/响应的内容，并根据其中的信息来完成数据的转发。也就是说，如果 HTTP 请求中不包含 "Host"header，则有可能在设置了 HTTP 代理的情况下无法与 HTTP 服务器建立连接。
- HTTP 代理时的 HTTPS/HTTP2 请求，与 HTTP 服务器建立通过 HTTP 代理的隧道连接。HTTP 代理不再解析传输的数据，仅仅完成数据转发的功能。此时 HTTP 代理的功能退化为如同 SOCKS 代理类似。
- 设置了代理类时，HTTP 的服务器的[域名解析](https://cloud.tencent.com/product/cns?from=10680)会交给代理服务器执行。其中设置了 HTTP 代理时，会对 HTTP代理的域名做域名解析。

#### 2.2.4 协议建立 establishProtocol

```kotlin
private fun establishProtocol(
    connectionSpecSelector: ConnectionSpecSelector,
    pingIntervalMillis: Int,
    call: Call,
    eventListener: EventListener
) {
    //如果不是ssl
    if (route.address.sslSocketFactory == null) {
        if (Protocol.H2_PRIOR_KNOWLEDGE in route.address.protocols) {
            socket = rawSocket
            protocol = Protocol.H2_PRIOR_KNOWLEDGE
            startHttp2(pingIntervalMillis)
            return
        }
        socket = rawSocket
        protocol = Protocol.HTTP_1_1
        return
    }

    eventListener.secureConnectStart(call)
    //如果是sll
    connectTls(connectionSpecSelector)

    eventListener.secureConnectEnd(call, handshake)
    
    //如果是HTTP2
    if (protocol === Protocol.HTTP_2) {
        startHttp2(pingIntervalMillis)
    }
}
```


不管是建立隧道连接，还是建立普通连接，都少不了建立协议这一步骤，这一步是建立好 TCP 连接之后，而在该TCP 能被拿来收发数据之前执行的**。它主要为了数据的加密传输做一些初始化，比如 TCL 握手，HTTP/2 的协商。**


1. 对于加密的数据传输，创建TLS连接。对于明文传输，则设置 protocol 和 socket。socket 直接指向应用层，如 HTTP 或 HTTP/2，交互的Socket。
- 对于明文传输没有设置 HTTP 代理的 HTTP 请求，它是与 HTTP 服务器之间的 TCP socket。
- 对于加密传输没有设置 HTTP 代理服务器的 HTTP 或 HTTP2 请求，它是与 HTTP 服务器之间的SSLSocket。 
- 对于加密传输设置了 HTTP 代理服务器的 HTTP 或 HTTP2 请求，它是与 HTTP 服务器之间经过代理服务器的SSLSocket，一个隧道连接； 
- 对于加密传输设置了 SOCKS 代理的 HTTP 或 HTTP2 请求，它是一条经过了代理服务器的 SSLSocket 连接。
2. 对于 HTTP/2，通过 new 一个 Http2Connection.Builder 会建立 HTTP/2 连接 Http2Connection，然后执行 http2Connection.start() 和服务器建立协议。

我们先来看下建立 TLS 连接的 connectTls() 方法：

```kotlin
private fun connectTls(connectionSpecSelector: ConnectionSpecSelector) {
    val address = route.address
    val sslSocketFactory = address.sslSocketFactory
    var success = false
    var sslSocket: SSLSocket? = null
    try {
        // Create the wrapper over the connected socket.
        sslSocket = sslSocketFactory!!.createSocket(
            rawSocket, address.url.host, address.url.port, true /* autoClose */) as SSLSocket

        // Configure the socket's ciphers, TLS versions, and extensions.
        val connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket)
        if (connectionSpec.supportsTlsExtensions) {
            Platform.get().configureTlsExtensions(sslSocket, address.url.host, address.protocols)
        }

        // Force handshake. This can throw!
        sslSocket.startHandshake()
        // block for session establishment
        val sslSocketSession = sslSocket.session
        val unverifiedHandshake = sslSocketSession.handshake()

        // Verify that the socket's certificates are acceptable for the target host.
        if (!address.hostnameVerifier!!.verify(address.url.host, sslSocketSession)) {
            val peerCertificates = unverifiedHandshake.peerCertificates
            if (peerCertificates.isNotEmpty()) {
                val cert = peerCertificates[0] as X509Certificate
                throw SSLPeerUnverifiedException("""
              |Hostname ${address.url.host} not verified:
              |    certificate: ${CertificatePinner.pin(cert)}
              |    DN: ${cert.subjectDN.name}
              |    subjectAltNames: ${OkHostnameVerifier.allSubjectAltNames(cert)}
              """.trimMargin())
            } else {
                throw SSLPeerUnverifiedException(
                    "Hostname ${address.url.host} not verified (no certificates)")
            }
        }

        val certificatePinner = address.certificatePinner!!

            handshake = Handshake(unverifiedHandshake.tlsVersion, unverifiedHandshake.cipherSuite,
                                  unverifiedHandshake.localCertificates) {
                certificatePinner.certificateChainCleaner!!.clean(unverifiedHandshake.peerCertificates,
                                                                  address.url.host)
            }

        // Check that the certificate pinner is satisfied by the certificates presented.
        certificatePinner.check(address.url.host) {
            handshake!!.peerCertificates.map { it as X509Certificate }
        }

        // Success! Save the handshake and the ALPN protocol.
        val maybeProtocol = if (connectionSpec.supportsTlsExtensions) {
            Platform.get().getSelectedProtocol(sslSocket)
        } else {
            null
        }
        socket = sslSocket
        source = sslSocket.source().buffer()
        sink = sslSocket.sink().buffer()
        protocol = if (maybeProtocol != null) Protocol.get(maybeProtocol) else Protocol.HTTP_1_1
        success = true
    } finally {
        if (sslSocket != null) {
            Platform.get().afterHandshake(sslSocket)
        }
        if (!success) {
            sslSocket?.closeQuietly()
        }
    }
}
```

TLS 连接是对原始 TCP 连接的一个封装，以及听过 TLS 握手，及数据收发过程中的加解密等功能。在Java中，用SSLSocket 来描述。上面建立的TLS连接的过程大体为：  
1、用 SSLSocketFactory 基于原始的 TCP Socket，创建一个 SSLSocket。  
2、配置 SSLSocket。  
3、在前面选择的 ConnectionSpec 支持 TLS 扩展参数时，配置 TLS 扩展参数。  
4、启动 TLS 握手  
5、TLS 握手完成之后，获取证书信息。  
6、对 TLS 握手过程中传回来的证书进行验证。  
7、在前面选择的 ConnectionSpec 支持 TLS 扩展参数时，获取 TLS 握手过程中顺便完成的协议协商过程所选择的协议。这个过程主要用于 HTTP/2 的 ALPN 扩展。  
8、OkHttp 主要使用 Okio 来做 IO 操作，这里会基于前面获取到 SSLSocket 创建于执行的 IO 的BufferedSource 和 BufferedSink 等，并保存握手信息以及所选择的协议。  

至此连接已经建立连接已经结束了。

#### 2.3.4 connect 后的操作
在 connect 后，会最后一次尝试查找可以复用的连接，只有在我们尝试到同一主机的多个并发连接时才会发生。
如果找不到，则把新创建的连接存入连接缓存池中。

```kotlin
synchronized(connectionPool) {
    connectingConnection = null
    //最后一次尝试查找可以复用的连接，只有在我们尝试到同一主机的多个并发连接时才会发生。
    if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
        // We lost the race! Close the connection we created and return the pooled connection.
        result!!.noNewExchanges = true
        socket = result!!.socket()
        result = call.connection

        // It's possible for us to obtain a coalesced connection that is immediately unhealthy. In
        // that case we will retry the route we just successfully connected with.
        nextRouteToTry = selectedRoute
    } else {
        //找不到就把新建的连接放进连接池中
        connectionPool.put(result!!)
        call.acquireConnectionNoEvents(result!!)
    }
}
```


### 2.3 判断连接是否健康 isHealthy

```kotlin
  fun isHealthy(doExtensiveChecks: Boolean): Boolean {
    val nowNs = System.nanoTime()

    val socket = this.socket!!
    val source = this.source!!
    if (socket.isClosed || socket.isInputShutdown || socket.isOutputShutdown) {
      return false
    }

    val http2Connection = this.http2Connection
    if (http2Connection != null) {
      return http2Connection.isHealthy(nowNs)
    }

    val idleDurationNs = nowNs - idleAtNs
    if (idleDurationNs >= IDLE_CONNECTION_HEALTHY_NS && doExtensiveChecks) {
      return socket.isHealthy(source)
    }

    return true
  }
```

参数 doExtensiveChecks 表示是否需要额外检查。

- **socket 已经关闭 **或 **输入流关闭 **或** 输出流关闭** 则不健康
- 如果是 HTTP/2 连接，如果 HTTP/2 连接关闭 则不健康
- 如果要额外检查，则调用 socket 的 isHealthy 方法检查是否健康

### 2.4 ExchangeCodec
经过上面流程后，找到了一个健康可用的 RealConnection 后，通过 newCodec 方法创建一个 ExchangeCodec：

```kotlin
  internal fun newCodec(client: OkHttpClient, chain: RealInterceptorChain): ExchangeCodec {
    val socket = this.socket!!
    val source = this.source!!
    val sink = this.sink!!
    val http2Connection = this.http2Connection

    return if (http2Connection != null) {
      Http2ExchangeCodec(client, this, chain, http2Connection)
    } else {
      socket.soTimeout = chain.readTimeoutMillis()
      source.timeout().timeout(chain.readTimeoutMillis.toLong(), MILLISECONDS)
      sink.timeout().timeout(chain.writeTimeoutMillis.toLong(), MILLISECONDS)
      Http1ExchangeCodec(client, this, source, sink)
    }
  }
```

里面主要是判断是否是 HTTP/2，如果是 HTTP/2 则 new 一个 Http2Codec。如果不是 HTTP/2 则 new 一个Http1Codec。


继续查看 [OkHttp 分析（三）](https://www.yuque.com/espoir/urcfma/eg4un1)
