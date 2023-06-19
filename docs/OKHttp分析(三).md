# 连接池 ConnectionPool

 管理 http 和 http/2 的链接，以便减少网络请求延迟。同一个 address 将共享同一个 connection。该类实现了复用连接的目标。

ConnectionPool 在 OkHttpClient 中创建，也可以在里面配置，ConnectionPool 的实现都桥接到 **RealConnectionPool** 去实现：

```kotlin
class ConnectionPool internal constructor(
  internal val delegate: RealConnectionPool
) {
  constructor(
    maxIdleConnections: Int,
    keepAliveDuration: Long,
    timeUnit: TimeUnit
  ) : this(RealConnectionPool(
      taskRunner = TaskRunner.INSTANCE,
      maxIdleConnections = maxIdleConnections,
      keepAliveDuration = keepAliveDuration,
      timeUnit = timeUnit
  ))

  constructor() : this(5, 5, TimeUnit.MINUTES)

  /** Returns the number of idle connections in the pool. */
  fun idleConnectionCount(): Int = delegate.idleConnectionCount()

  /** Returns total number of connections in the pool. */
  fun connectionCount(): Int = delegate.connectionCount()

  /** Close and remove all idle connections in the pool. */
  fun evictAll() {
    delegate.evictAll()
  }
}
```

ConnectionPool 在创建时，指定了 3 个参数，分别是

- **最大空闲连接数 maxIdleConnections，默认是 5.**
- **存活时间 keepAliveDuration，单位是分钟。**

**
通过这个构造器我们知道了这个连接池最多维持5个连接，且每个链接最多活5分钟。 
 **maxIdleConnections 和 keepAliveDurationNs 是清理淘汰连接的的指标**，**这里需要说明的是maxIdleConnections 是值每个地址上最大的空闲连接数。所以 OkHttp 只是限制与同一个远程服务器的空闲连接数量，对整体的空闲连接并没有限制。**
**
# RealConnectionPool

上面创建 ConnectionPool 实际上是创建 RealConnectionPool：

```kotlin
class RealConnectionPool(
  taskRunner: TaskRunner,
  /** The maximum number of idle connections for each address. */
  private val maxIdleConnections: Int,
  keepAliveDuration: Long,
  timeUnit: TimeUnit
) {
  private val keepAliveDurationNs: Long = timeUnit.toNanos(keepAliveDuration)

  private val cleanupQueue: TaskQueue = taskRunner.newQueue()
  private val cleanupTask = object : Task("$okHttpName ConnectionPool") {
    override fun runOnce() = cleanup(System.nanoTime())
  }

  private val connections = ArrayDeque<RealConnection>()
  //...
}
```

从 RealConnectionPool 的变量可知，它包含了**清理任务**和 **RealConnection 队列**。所以我们先看看队列的基本操作。

## 查找
由上篇分析可以知道，RealConnectionPool 的查找操作并不是简单的从队列中找出某个 value，而是通过 **callAcquirePooledConnection** 方法查找出可以复用的连接：

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

callAcquirePooledConnection 方法已经在上篇分析过，这里注意的是如果查找到了，会通过 RealCall 的 acquireConnectionNoEvents 方法更新当前 RealCall 中的连接。

## put(connection: RealConnection) 添加

```kotlin
private val cleanupQueue: TaskQueue = taskRunner.newQueue()
private val cleanupTask = object : Task("$okHttpName ConnectionPool") {
    override fun runOnce() = cleanup(System.nanoTime())
}

fun put(connection: RealConnection) {
    this.assertThreadHoldsLock()

    connections.add(connection)
    // 回收连接池中无用连接的操作
    cleanupQueue.schedule(cleanupTask)
}
```

put 方法首先就是往队列中添加当前连接，然后会调用 **cleanupQueue.schedule(cleanupTask) 执行清理操作。**
**cleanupQueue 是 TaskQueue 对象，由 TaskRunner#newQueue 方法创建，TaskRunner 在 ConnectionPool 中通过 TaskRunner.INSTANCE 创建。

```kotlin
companion object {
    @JvmField
    val INSTANCE = TaskRunner(RealBackend(threadFactory("$okHttpName TaskRunner", daemon = true)))
    //...
}

class RealBackend(threadFactory: ThreadFactory) : Backend {
    private val executor = ThreadPoolExecutor(
        0, // corePoolSize.
        Int.MAX_VALUE, // maximumPoolSize.
        60L, TimeUnit.SECONDS, // keepAliveTime.
        SynchronousQueue(),
        threadFactory
    )
    //...
}
```

可以看到，这是一个线程池，每个线程池最多只能运行一个线程，并且这个线程池允许被垃圾回收。
通过追踪 cleanupQueue 的 schedule 方法可知，这里其实是线程池执行 Runnable 并最终回调 cleanupTask 的 runOnce 方法，而 runOnce 又调用了 cleanup 方法。

### cleanup

```kotlin
for (connection in connections) {
    //如果连接正在使用，继续搜索
    if (pruneAndGetAllocationCount(connection, now) > 0) {
        inUseConnectionCount++
        continue
    }
    idleConnectionCount++
    //如果空闲的话，connection.idleAtNs 会不断地减去5分钟，所以空闲连接越多，idleDurationNs 的值越大
    val idleDurationNs = now - connection.idleAtNs 
    if (idleDurationNs > longestIdleDurationNs) {
        longestIdleDurationNs = idleDurationNs
        longestIdleConnection = connection
    }
}
```

在 cleanup 中，
1 、首先是通过 for 循环统计**不空闲的连接数量** **inUseConnectionCount**。
2、然后查找最长空闲时间的连接以及对应空闲时长，判断是否超出最大空闲连接数 **maxIdleConnections **或者超过最大空闲时间 **keepAliveDurationNs**，满足其一则清除最长空闲时长的连接。如果不满足清理条件，则返回一个对应等待时间。

#### pruneAndGetAllocationCount

```kotlin
 private fun pruneAndGetAllocationCount(connection: RealConnection, now: Long): Int {
    val references = connection.calls
    var i = 0
    while (i < references.size) {
      val reference = references[i]

      if (reference.get() != null) {
        i++
        continue
      }
      
      //...
      references.removeAt(i)
      connection.noNewExchanges = true

      //如果列表为空则说明此连接没有被引用了，则返回0，表示此连接是空闲连接
      if (references.isEmpty()) {
        connection.idleAtNs = now - keepAliveDurationNs
        return 0
      }
    }

    return references.size
  }
```

该方法是查找空闲连接数用的，方法逻辑：

- 遍历 references 即 calls 列表，如果不为空，计数器加一，并跳出当前循环进行下一次循环。
- 如果为空，则将它从 calls 列表中删除，修改 noNewExchanges 变量
- 如果列表为空了证明此连接没有被引用了，则返回 0，并且更新空闲时间 idleAtNs 为当前时间减去 5 分钟。
- 否则返回列表长度
- 如果空闲的话，connection.idleAtNs 会不断地减去 5 分钟

**
**断方法和 leakcanary 的原理类似。**


RealConnection 的 calls：

```kotlin
val calls = mutableListOf<Reference<RealCall>>()

#RealCall
fun acquireConnectionNoEvents(connection: RealConnection) {
    connectionPool.assertThreadHoldsLock()

    check(this.connection == null)
    this.connection = connection
    connection.calls.add(CallReference(this, callStackTrace))
}

internal class CallReference(
    referent: RealCall,
    val callStackTrace: Any?
) : WeakReference<RealCall>(referent)
```

calls 是一个保存了 Call 的弱引用列表，它在 RealCall 的 **acquireConnectionNoEvents** 方法中被添加。而 **acquireConnectionNoEvents** 方法在创建完新的连接 RealConnection 并添加到连接池后调用：

```kotlin
 private fun findConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean
  ): RealConnection {
    //...
    synchronized(connectionPool) 
      //最后一次尝试进行连接合并，只有在我们尝试到同一主机的多个并发连接时才会发生。
      if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
        //..
      } else {
        connectionPool.put(result!!)
        call.acquireConnectionNoEvents(result!!)
      }
    }
    //...
    return result!!
  }
```

在 callAcquirePooledConnection 方法中，如果连接可以复用，也会调用 **acquireConnectionNoEvents** 方法：


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
      call.acquireConnectionNoEvents(connection)  //<-
      return true
    }
    return false
  }
```

在看 for 循环中的这段：

```kotlin
val idleDurationNs = now - connection.idleAtNs 
if (idleDurationNs > longestIdleDurationNs) {
    longestIdleDurationNs = idleDurationNs
    longestIdleConnection = connection
}
```

由上面分析可得，如果空闲连接数越多，connection.idleAtNs 会不断地减去 5 分钟，**所以这里 idleDurationNs 的值会随着空闲数越多，值越大。**
当值大于 longestIdleDurationNs（默认值 Long.MIN_VALUE）时，更新 longestIdleDurationNs 为 idleDurationNs，longestIdleConnection 为当前不空闲的 connection。
所以这段代码计算的是**最长空闲时间 longestIdleDurationNs 以及最长空闲的那个连接 longestIdleConnection**

#### when 判断
在 cleanup 中，for 循环结束后，到了 when 判断逻辑：


```kotlin
when {
    longestIdleDurationNs >= this.keepAliveDurationNs
    || idleConnectionCount > this.maxIdleConnections -> {
        connections.remove(longestIdleConnection)    //就将此泄漏连接从Deque中移除
        if (connections.isEmpty()) cleanupQueue.cancelAll()
    }
    idleConnectionCount > 0 -> {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs
    }
    inUseConnectionCount > 0 -> {
        // 全部都是活跃的连接，5分钟后再次清理
        return keepAliveDurationNs
    }
    else -> {
        // No connections, idle or in use.
        return -1  //没有任何连接，跳出循环
    }
}
```

- 如果最长空闲时间超过了 5 分钟，或者空闲数量超过了 5 个的话，会将空闲连接 longestIdleConnection 从队列中移除
- 否则如果空闲数量大于 0，但没超过 5 个，用 5 分钟减去最长空闲时间，返回时间差。
- 如果空闲数量等于 0，不空闲数量大于 0，则代表的是全部都是活跃的连接，则返回 5 分钟。
- 否则返回 -1，表示没有任何连接，跳出循环。


## initExchange
到这里 exchangeFinder 的 find 方法算是分析得差不多了，现在重新看看 RealCall 的 initExchange 方法以及  ConnectInterceptor 的代码。

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

在创建完 ExchangeCodec 后，再新建 Exchange 对象，并且保存起来。在 ConnectInterceptor 中，通过 copy 方法重新创建了 RealInterceptorChain 对象，并且把创建的 Exchange 传进去，再执行下一个拦截器。


# CallServerInterceptor

它是最后一个拦截器，CallServerInterceptor 拦截器的功能就是负责与服务器建立 Socket连接，并且创建了一个 HttpStream， 它包括通向服务器的输入流和输出流。使用 HttpStream与服务器进行数据的读写操作，即发起网络请求接收网络响应。

```kotlin
class CallServerInterceptor(private val forWebSocket: Boolean) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.exchange!!
    val request = realChain.request
    val requestBody = request.body
    val sentRequestMillis = System.currentTimeMillis()
    exchange.writeRequestHeaders(request)
    //...
  }
```

在 CallServerInterceptor 中，首先拿到一些变，其中 exchange（Exchange） 就是上面得到的。这里主要也是执行它里面的方法，而在 Exchange 内部，主要也是调用 codec（ExchangeCodec）方法，比如：

```kotlin
fun writeRequestHeaders(request: Request) {
    try {
        eventListener.requestHeadersStart(call)
        codec.writeRequestHeaders(request)
        eventListener.requestHeadersEnd(call, request)
    } catch (e: IOException) {
        eventListener.requestFailed(call, e)
        trackFailure(e)
        throw e
    }
}
```

ExchangeCodec 主要由这么几个方法：

```kotlin
//写入请求头部信息的方法
void writeRequestHeaders(Request request) throws IOException;

//写入请求的body信息
Sink createRequestBody(Request request, long contentLength);
 
//读取响应的头信息
Response.Builder readResponseHeaders(boolean expectContinue) throws IOException;

//读取响应体body
ResponseBody openResponseBody(Response response) throws IOException;

//标识请求过程完成
void finishRequest() throws IOException;
```

它的实现类就是上面分析过的 Http1ExchangeCodec 或者 Http2ExchangeCodec 。


**CallServerInterceptor 流程：**

- 写入请求头信息：httpCodec.writeRequestHeaders(request)；
- 有请求体的情况下，写入请求体信息：httpCodec.createRequestBody(request, contentLength)；
- 结束请求：httpCodec.finishRequest()；
- 读取响应头信息：httpCodec.readResponseHeaders(false)；
- 获取响应体信息输入流：httpCodec.openResponseBody(response)；
- 往上一级 ConnectInterceptor 返回一个网络请求回来的 Response；

# OkHttp 中https 使用ip直连会存在问题吗？
## 什么是 Dns?

在网络的世界中，每个有效的域名背后都有为其提供服务的服务器，而我们网络通信的首要条件，就是知道服务器的 IP 地址。
但是记住域名（网址）肯定是比记住 IP 地址简单。如果有某种方法，可以通过域名，查到其提供服务的服务器 IP 地址，那就非常方便了。这里就需要用到 DNS 服务器以及 DNS 解析。
**DNS（Domain Name System），它的作用就是根据域名，查出对应的 IP 地址**，它是 HTTP 协议的前提。只有将域名正确的解析成 IP 地址后，后面的 HTTP 流程才可以继续进行下去。

## 优点
使用Interceptor做ip直连，则会存在以下优点：

- 对 Dns 的控制偏上层，可更加细化，控制灵活。
- 容灾处理更容易。


## 缺点

- 一切跟域名有关的处理全部失效，具体有：
- 在 Https 下处理 SSL 证书会出现校验问题
- ip 访问时出现 Cookie 校验问题。

# OkHttp 代理与路由机制
在之前分析中，可以留意到在查找健康连接** findConnection **方法中，出现了路由选择逻辑，主要的类有 RouteSelector。

## Route
先了解什么是 Route：

```kotlin
class Route(
  @get:JvmName("address") val address: Address,
  @get:JvmName("proxy") val proxy: Proxy,
  @get:JvmName("socketAddress") val socketAddress: InetSocketAddress
) {
    //...
}
```

Route 中有三个变量，分别是 Address，Proxy 和 InetSocketAddress，它是一个用于描述一条路由的类。
Proxy 是代理服务器信息，它是由 Java 原生提供的。
InetSocketAddress 是连接目标地址，描述一条路由，它是由 Java 原生提供的。

### Proxy

```kotlin
public class Proxy {
    public enum Type {
        DIRECT, // 表示不使用代理
        HTTP,   // HTTP代理
        SOCKS   // SOCKS代理
    };
    private Type type;
    private SocketAddress sa;
    // ...
}
```

它是一个用于描述代理服务器的类，主要包含了代理协议的类型以及代理服务器对应的 SocketAddress 类，有以下三种类型：

- **DIRECT**：不使用代理
- **HTTP**：HTTP 代理
- **SOCKS**：SOCKS 代理

### RouteSelector
在 findConnection 方法中，创建了 RouteSelector 对象，并且调用 next() 方法进行路由选择：


```kotlin
var newRouteSelection = false
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

RouteSelecter 是一个负责负责管理路由信息，并辅助选择路由的类。它主要有三个职责：

- 收集可用的路由
- 选择可用的路由
- 维护连接失败路由信息

 
#### 代理的收集
在 RouteSelecter 的构造方法中，调用了 resetNextProxy 方法：

```kotlin
init{
   resetNextProxy(address.url, address.proxy)
}

private fun resetNextProxy(url: HttpUrl, proxy: Proxy?) {
    fun selectProxies(): List<Proxy> {
        //若用户有设定代理，使用用户设置的代理
        if (proxy != null) return listOf(proxy)
		//如果URI缺少主机（如“ http：// </”中的内容）则不要调用ProxySelector。
        val uri = url.toUri()
        if (uri.host == null) return immutableListOf(Proxy.NO_PROXY)
		//尝试每个ProxySelector选项，直到一个连接成功。获取代课列表
        val proxiesOrNull = address.proxySelector.select(uri)
        if (proxiesOrNull.isNullOrEmpty()) return immutableListOf(Proxy.NO_PROXY)

        return proxiesOrNull.toImmutableList()
    }

    eventListener.proxySelectStart(call, url)
    proxies = selectProxies()
    nextProxyIndex = 0
    eventListener.proxySelectEnd(call, url, proxies)
}
```

- 首先检查了一下我们的 address 中有没有用户设定的代理（通过 OkHttpClient 传入），若有用户设定的代理，则直接使用用户设定的代理。
- 若用户没有设定的代理，则尝试使用 ProxySelector.select 方法来获取代理列表。这里的 ProxySelector 也可以通过 OkHttpClient 进行设置，默认情况下会使用系统默认的 ProxySelector 来获取系统配置中的代理列表。

 
#### 选择可用路由
在代理选择成功之后，会进行可用路由的选择工作:

```kotlin
 operator fun next(): Selection {
    if (!hasNext()) throw NoSuchElementException()

    val routes = mutableListOf<Route>()
    while (hasNextProxy()) {
      //延迟的路线总是最后尝试。 例如，如果我们有2个代理，并且应该推迟proxy1的所有路由，
      //我们将移至proxy2。 只有在用尽所有好的路线后，我们才会尝试推迟的路线。
      //即优先使用正常的路由
      val proxy = nextProxy()
      for (inetSocketAddress in inetSocketAddresses) {
        val route = Route(address, proxy, inetSocketAddress)
        if (routeDatabase.shouldPostpone(route)) {
          postponedRoutes += route
        } else {
          routes += route
        }
      }

      if (routes.isNotEmpty()) {
        break
      }
    }
	// 若找不到正常的路由，则只能采用连接失败的路由
    if (routes.isEmpty()) {
      // We've exhausted all Proxies so fallback to the postponed routes.
      routes += postponedRoutes
      postponedRoutes.clear()
    }

    return Selection(routes)
  }
```

在 next 方法中，通过遍历 inetSocketAddresses 寻找正常路由添加到 list 中，如果 list 为空，则代表找不到正常路由，只能用连接失败的路由。

**RouteDatabase 路由数据库**，里面用一个 list 维护着失败的路由：

```kotlin
class RouteDatabase {
  private val failedRoutes = mutableSetOf<Route>()
  //失败时添加进去list
  @Synchronized fun failed(failedRoute: Route) {
    failedRoutes.add(failedRoute)
  }
  //成功时从list中移除
  @Synchronized fun connected(route: Route) {
    failedRoutes.remove(route)
  }
  //查找
  @Synchronized fun shouldPostpone(route: Route): Boolean = route in failedRoutes
}
```

**nextProxy() 方法**
**
```kotlin
private fun nextProxy(): Proxy {
    if (!hasNextProxy()) {
        throw SocketException(
            "No route to ${address.url.host}; exhausted proxy configurations: $proxies")
    }
    val result = proxies[nextProxyIndex++]
    resetNextInetSocketAddress(result)
    return result
}

private fun resetNextInetSocketAddress(proxy: Proxy) {
    // Clear the addresses. Necessary if getAllByName() below throws!
    val mutableInetSocketAddresses = mutableListOf<InetSocketAddress>()
    inetSocketAddresses = mutableInetSocketAddresses

    val socketHost: String
    val socketPort: Int
    // 若是 DIRECT 及 SOCKS 代理，则向原目标的 host 和 port 进行请求
    if (proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.SOCKS) {
        socketHost = address.url.host
        socketPort = address.url.port
    } else {
        // 若是 HTTP 代理，通过代理的地址请求代理服务器的 host
        val proxyAddress = proxy.address()
        require(proxyAddress is InetSocketAddress) {
            "Proxy.address() is not an InetSocketAddress: ${proxyAddress.javaClass}"
        }
        socketHost = proxyAddress.socketHost
        socketPort = proxyAddress.port
    }
   
    if (socketPort !in 1..65535) {
        throw SocketException("No route to $socketHost:$socketPort; port is out of range")
    }
     // 代理类型为 SOCKS 则直接填入原目标的 host 和 port（因为不需要 DNS 解析）
    if (proxy.type() == Proxy.Type.SOCKS) {
        mutableInetSocketAddresses += InetSocketAddress.createUnresolved(socketHost, socketPort)
    } else {
        // HTTP 和 DIRECT 代理，进行 DNS 解析后填入 dns 解析后的 ip 地址和端口
        eventListener.dnsStart(call, socketHost) //回调 dnsStart
        // 用户可重写 lookup 方法自定义 Dns
        val addresses = address.dns.lookup(socketHost)
        if (addresses.isEmpty()) {
            throw UnknownHostException("${address.dns} returned no addresses for $socketHost")
        }
		//回调 dnsEnd
        eventListener.dnsEnd(call, socketHost, addresses)
        for (inetAddress in addresses) {
            mutableInetSocketAddresses += InetSocketAddress(inetAddress, socketPort)
        }
    }
}
```

它主要就是在之前收集的代理列表中获取下一个代理的信息，并且调用 **resetNextInetSocketAddress** 方法根据代理协议获取对应的 **Address** 相关信息填入 **inetSocketAddresses** 中。

- 若是 DIRECT 及 SOCKS 代理，则向原目标的 host 和 port 进行请求
- 若是 HTTP 代理，通过代理的地址请求代理服务器的 host
- 代理类型为 SOCKS 则直接填入原目标的 host 和 port（因为不需要 DNS 解析）
- HTTP 和 DIRECT 代理，进行 DNS 解析后填入 dns 解析后的 ip 地址和端口
