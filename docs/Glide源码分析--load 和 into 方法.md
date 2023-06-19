# load
load 方法在 RequestManager 这个类里面，load 方法有多个重载，具体可以看 ModelTypes 接口。RequestManager 除了 load 方法外，还有两个 downloadOnly 和 download 方法：

```java
RequestBuilder<Drawable> load(@Nullable Bitmap bitmap);
RequestBuilder<Drawable> load(@Nullable Drawable drawable);
RequestBuilder<Drawable> load(@Nullable String string);
RequestBuilder<Drawable> load(@Nullable Uri uri);
RequestBuilder<Drawable> load(@Nullable File file);
RequestBuilder<Drawable> load(@RawRes @DrawableRes @Nullable Integer resourceId);
RequestBuilder<Drawable> load(@Nullable URL url);
RequestBuilder<Drawable> load(@Nullable byte[] model);
RequestBuilder<Drawable> load(@Nullable Object model);

RequestBuilder<File> downloadOnly();
RequestBuilder<File> download(@Nullable Object model);
```

下面我们挑一个 load 方法来看：

```java
public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
}

public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}

public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

load 方法首先会调用 asDrawable 方法创建 RequestBuilder，然后再调用 RequestBuilder 的 load 方法。

RequestBuilder 是一个通用类，用来构建请求和设置处理通用资源类型的设置选项和启动负载。（例如设置 RequestOption、缩略图、加载失败占位图等等。） 

RequestManager 中的 load 方法都对应着 RequestBulider 的 load 方法。

```java
public RequestBuilder<TranscodeType> load(@Nullable String string) {
  return loadGeneric(string);
}

private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
  this.model = model;
  isModelSet = true;
  return this;
}
```

一般来说 load 方法之后就是调用 into 方法设置 ImageView 或者 Target，into 方法中最后会创建 Request，并启动。

# into

```java
private <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    BaseRequestOptions<?> options,
    Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    if (!isModelSet) {
        throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    Request request = buildRequest(target, targetListener, options, callbackExecutor);
	//获取以前的请求
    Request previous = target.getRequest();
    //两个请求是否相同
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
        //释放
        request.recycle();
        //如果以前的请求不是在运行
        if (!Preconditions.checkNotNull(previous).isRunning()) {
			//开始请求
            previous.begin();
        }
        return target;
    }
   
    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);

    return target;
}
```

load 方法后的 into 最后都会调用上面代码所示的 into 方法，在这个方法内，通过 buildRequest 创建了 Request 对象。

into 有几个重载，这里拿常用的需要传入 ImageView 的重载来看。

**在 into 方法里，有一个参数 target，当我们使用时，会先根据 Target 类型创建不同的 Target，然后 RequestBuilder 将这个 target 当做参数创建 Request 对象，Request 与 Target 就是这样关联起来的。**
 
而 ImageView 对应的 target 由 ImageViewTargetFactory 负责创建，继承于 ViewTarget。从上面方法中可以看到。into 会先从 target 中取出缓存的 request，如果存在并且等同于当前的 request，则释放当前的 request ，执行缓存的 request 的 begin 方法。否则通过 target.setRequest(request); 存起来。

在 ViewTarget 中，setRequest 其实就是调用 View 的 setTag 方法，这也是为什么当你在用 Glide 时不能给 View 设置 tag 的原因。
```java
public void setRequest(@Nullable Request request) {
  setTag(request);
}

private void setTag(@Nullable Object tag) {
  if (tagId == null) {
    isTagUsedAtLeastOnce = true;
    view.setTag(tag);
  } else {
    view.setTag(tagId, tag);
  }
}

public Request getRequest() {
  Object tag = getTag();
  Request request = null;
  if (tag != null) {
    if (tag instanceof Request) {
      request = (Request) tag;
    } else {
      throw new IllegalArgumentException(
        "You must not call setTag() on a view Glide is targeting");
    }
  }
  return request;
}
```

**requestManager.track(target, request);**
在 into 方法的最后一行，看 track 做了什么：

```java
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
  targetTracker.track(target);
  requestTracker.runRequest(request);
}
```

首先调用 TargetTracker 的 track 方法把 target 存起来。TargetTracker 实现了 LifecycleListener，它的作用是回调 target 的生命周期方法：

```java
public final class TargetTracker implements LifecycleListener {
  private final Set<Target<?>> targets =
      Collections.newSetFromMap(new WeakHashMap<Target<?>, Boolean>());

  public void track(@NonNull Target<?> target) {
    targets.add(target);
  }

  public void untrack(@NonNull Target<?> target) {
    targets.remove(target);
  }

  @Override
  public void onStart() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onStart();
    }
  }

  @Override
  public void onStop() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onStop();
    }
  }

  @Override
  public void onDestroy() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onDestroy();
    }
  }
  //...
}
```

TargetTracker 在 RequestManager 中调用：

```java
public class RequestManager implements LifecycleListener,ModelTypes<RequestBuilder<Drawable>> {
  //...
  @GuardedBy("this")
  private final TargetTracker targetTracker = new TargetTracker();
  //...
  @Override
  public synchronized void onStart() {
    resumeRequests();
    targetTracker.onStart();
  }

  /**
   * Lifecycle callback that unregisters for connectivity events (if the
   * android.permission.ACCESS_NETWORK_STATE permission is present) and pauses in progress loads.
   */
  @Override
  public synchronized void onStop() {
    pauseRequests();
    targetTracker.onStop();
  }

  /**
   * Lifecycle callback that cancels all in progress requests and clears and recycles resources for
   * all completed requests.
   */
  @Override
  public synchronized void onDestroy() {
    targetTracker.onDestroy();
    //...
  }
}
```

然后看 RequestTracker#runRequest 方法，判断Glide当前是不是处理暂停状态，如果不是暂停状态就调用Request的begin()方法来执行Request，否则的话就先将Request添加到待执行队列里面，等暂停状态解除了之后再执行。

```java
public void runRequest(@NonNull Request request) {
  requests.add(request);
  if (!isPaused) {
    request.begin();
  } else {
    request.clear();
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Paused, delaying request");
    }
    pendingRequests.add(request);
  }
}
```



# Request
![glide3.png](https://s2.loli.net/2023/06/19/fDQp2xs8LWkbYF3.png)

Request 是一个接口，加载资源的请求是基于这个接口的，它的实现类有三个：
![glide4.png](https://s2.loli.net/2023/06/19/r9MzWUNZCkAIXbs.png)

**SingleRequest**
 
这个类负责执行请求并将结果反映到 Target 上。在 RequestBuilder#buildRequest 方法中，如果按正常流程，创建的是 SingleRequest，下面看 SingleRequest#begin 方法：

```java
public synchronized void begin() {
    assertNotCallingCallbacks();
    stateVerifier.throwIfRecycled();
    startTime = LogTime.getLogTime();
    if (model == null) {
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        width = overrideWidth;
        height = overrideHeight;
      }
      // Only log at more verbose log levels if the user has set a fallback drawable, because
      // fallback Drawables indicate the user expects null models occasionally.
      int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
      onLoadFailed(new GlideException("Received null model"), logLevel);
      return;
    }

    if (status == Status.RUNNING) {
      throw new IllegalArgumentException("Cannot restart a running request");
    }

    //如果我们在完成之后重新启动（通常通过诸如notifyDataSetChanged之类的东西，
    //在相同的目标或视图中启动相同的请求），我们可以简单地使用上次检索的资源和大小，
    //并跳过获取新的大小 这意味着想要重新启动负载因为期望视图大小已更改的用户
    //需要在开始新加载之前明确清除视图或目标。
    if (status == Status.COMPLETE) {
      onResourceReady(resource, DataSource.MEMORY_CACHE);
      return;
    }
	//
    status = Status.WAITING_FOR_SIZE;
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      onSizeReady(overrideWidth, overrideHeight);
    } else {
      target.getSize(this);
    }

    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
        && canNotifyStatusChanged()) {
      target.onLoadStarted(getPlaceholderDrawable());
    }
    if (IS_VERBOSE_LOGGABLE) {
      logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    }
  }
```

代码如上，我们看下半段，首先会调用 isValidDimensions 去检查尺寸，如果宽高都大于 0 ，则调用 onSizeReady 方法，否则调用 target.getSize 方法。因为我们经常在 Activity#onCreate 方法中直接使用 Glide 方法，但此时的图片大小还未确定，所以调用 Request#begin 时并不会直接发起请求。所以接下来看 ViewTarget#getSize 方法：

```java
void getSize(@NonNull SizeReadyCallback cb) {
    int currentWidth = getTargetWidth();
    int currentHeight = getTargetHeight();
    if (isViewStateAndSizeValid(currentWidth, currentHeight)) {
        cb.onSizeReady(currentWidth, currentHeight);
        return;
    }

    // We want to notify callbacks in the order they were added and we only expect one or two
    // callbacks to be added a time, so a List is a reasonable choice.
    if (!cbs.contains(cb)) {
        cbs.add(cb);
    }
    if (layoutListener == null) {
        ViewTreeObserver observer = view.getViewTreeObserver();
        layoutListener = new SizeDeterminerLayoutListener(this);
        observer.addOnPreDrawListener(layoutListener);
    }
}
```

因为 SingleRequest 实现了 SizeReadyCallback 回调，所以当 getSize 完成后同样会回调 onSizeReady 方法。
可以看到，在 getSize 中，会给 View 注册 ViewTreeObserver.OnPreDrawListener 监听。在 SizeDeterminerLayoutListener 中，会回调 checkCurrentDimens ，再调用 notifyCbs，在 notifyCbs 里面回调 onSizeReady。

在 onSizeReady 中也不会直接去请求，而是调用了 Engine#load 方法加载，这个方法差不多有二十个参数，所以 onSizeReady 方法算是用来构建参数列表并且调用 Engine#load 方法的。

```java
public synchronized void onSizeReady(int width, int height) {
    //...
    loadStatus =
        engine.load(
            glideContext,
            model,
            requestOptions.getSignature(),
            this.width,
            this.height,
            requestOptions.getResourceClass(),
            transcodeClass,
            priority,
            requestOptions.getDiskCacheStrategy(),
            requestOptions.getTransformations(),
            requestOptions.isTransformationRequired(),
            requestOptions.isScaleOnlyOrNoTransform(),
            requestOptions.getOptions(),
            requestOptions.isMemoryCacheable(),
            requestOptions.getUseUnlimitedSourceGeneratorsPool(),
            requestOptions.getUseAnimationPool(),
            requestOptions.getOnlyRetrieveFromCache(),
            this,
            callbackExecutor);
    //...
  }
```

SingleRequest 实现了 ResourceCallback 接口，而在 load 方法中，倒数第二个参数正是传入 ResourceCallback。

```java
public interface ResourceCallback {
 
  void onResourceReady(Resource<?> resource, DataSource dataSource);

  void onLoadFailed(GlideException e);
}
```

加载完后会回调这两个方法，

Engine 负责管理请求以及活动资源、缓存等。主要关注 load 方法，这个方法主要做了如下几件事：

1. 通过请求构建 Key；
2. 从活动资源中获取资源，获取到则返回；
3. 从缓存中获取资源，获取到则直接返回；
4. 判断当前请求是否正在执行，是则直接返回；
5. 构建 EngineJob 与 DecodeJob 并执行。

下面看 Engine#load 方法:

```java
public synchronized <R> LoadStatus load(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);

    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return null;
    }

    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return null;
    }

    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Added to existing load", startTime, key);
      }
      return new LoadStatus(cb, current);
    }

    EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);

    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
            glideContext,
            model,
            key,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            onlyRetrieveFromCache,
            options,
            engineJob);

    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);

    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
  }
```

load 方法上半部分是跟缓存的存取有关的，这里先看下半部分：

**EngineJob**
这个主要用来执行 DecodeJob 以及管理加载完成的回调，各种监听器，没有太多其他的东西。

**DecodeJob**
DecodeJob 实现了 Runnable ，在 load 中，通过 start 方法实现图片请求。

```java
public synchronized void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    GlideExecutor executor = decodeJob.willDecodeFromCache()
        ? diskCacheExecutor
        : getActiveSourceExecutor();
    executor.execute(decodeJob);
}
```

start 方法的实现是通过一个线程池执行 decodeJob 这个 Runnable 。所以接下来看 DecodeJob 的 run 方法，
在 run 方法中，主体结构是一个 try catch finally。主要调用了 runWrapped 方法：

```java
private void runWrapped() {
    switch (runReason) {
        case INITIALIZE:
            stage = getNextStage(Stage.INITIALIZE);
            currentGenerator = getNextGenerator();
            runGenerators();
            break;
        case SWITCH_TO_SOURCE_SERVICE:
            runGenerators();
            break;
        case DECODE_DATA:
            decodeFromRetrievedData();
            break;
        default:
            throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
}

private Stage getNextStage(Stage current) {
    switch (current) {
        case INITIALIZE:
            return diskCacheStrategy.decodeCachedResource()
                ? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
        case RESOURCE_CACHE:
            return diskCacheStrategy.decodeCachedData()
                ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
        case DATA_CACHE:
            // Skip loading from source if the user opted to only retrieve the resource from cache.
            return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
        case SOURCE:
        case FINISHED:
            return Stage.FINISHED;
        default:
            throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
}

private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
        case RESOURCE_CACHE:
            return new ResourceCacheGenerator(decodeHelper, this);
        case DATA_CACHE:
            return new DataCacheGenerator(decodeHelper, this);
        case SOURCE:
            return new SourceGenerator(decodeHelper, this);
        case FINISHED:
            return null;
        default:
            throw new IllegalStateException("Unrecognized stage: " + stage);
    }
}
```

在 runWrapped 中，如果是第一次调用，会走 INITIALIZE , 然后调用 getNextStage，同样地，第一次调用会走 INITIALIZE，然后再走一遍 getNextStage，不过这时候传入的参数是 Stage.RESOURCE_CACHE，然后接下来再次调用 getNextStage，最后这个方法返回的是 Stage.SOURCE。

走完了 getNextStage，然后再走 getNextGenerator ，根据返回结果，getNextGenerator 中创建的是 SourceGenerator 实例，然后再调用 runGenerators。在 runGenerators 中，会调用 SourceGenerator#startNext 方法。

```java
public boolean startNext() {
    if (dataToCache != null) {
        Object data = dataToCache;
        dataToCache = null;
        cacheData(data);
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
        return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
        loadData = helper.getLoadData().get(loadDataListIndex++);
        if (loadData != null
            && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
                || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
            started = true;
            loadData.fetcher.loadData(helper.getPriority(), this);
        }
    }
    return started;
}
```

在这个方法里面，先看 loadData 这个对象，它是通过 DecodeHelper#getLoadData#get 方法获取的， getLoadData 方法返回的是一个 List：

List<ModelLoader<Object, ?>> modelLoaders = glideContext.getRegistry().getModelLoaders(model);

是通过 Registry#getModelLoaders 获取的，而 Registry 是在创建 Glide 实例的时候创建的。Registry 里面有各种请求和解析的方式。

ModelLoader 是一个接口，它的实现类有很多个，都是在 Glide 的构造函数中通过 Registry 的 append 方法创建并注册进去的。

```java
public <Model, Data> Registry append(
    @NonNull Class<Model> modelClass, @NonNull Class<Data> dataClass,
    @NonNull ModelLoaderFactory<Model, Data> factory) {
    modelLoaderRegistry.append(modelClass, dataClass, factory);
    return this;
}
```

其中有一个实现类是 HttpUriLoader 是这里用到的，loadData.fetcher.loadData 其实是调用了 **HttpUriLoader#DataFetcher#loadData **方法。

DataFetcher 是一个接口，重要的一个方法是 loadData，这个方法是加载数据用的。DataFetcher 有非常多个实现类，我们看其中一个叫做 HttpUrlFetcher 的，看名字应该是根据 url 实现图片加载的类，看他的 loadData 方法：

```java
public void loadData(@NonNull Priority priority,
                     @NonNull DataCallback<? super InputStream> callback) {
  long startTime = LogTime.getLogTime();
  try {
    InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
    callback.onDataReady(result);
  } catch (IOException e) {
    if (Log.isLoggable(TAG, Log.DEBUG)) {
      Log.d(TAG, "Failed to load data for url", e);
    }
    callback.onLoadFailed(e);
  } finally {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Finished http url fetcher fetch in " + LogTime.getElapsedMillis(startTime));
    }
  }
}

private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl,
                                          Map<String, String> headers) throws IOException {
  if (redirects >= MAXIMUM_REDIRECTS) {
    throw new HttpException("Too many (> " + MAXIMUM_REDIRECTS + ") redirects!");
  } else {
    // Comparing the URLs using .equals performs additional network I/O and is generally broken.
    // See http://michaelscharf.blogspot.com/2006/11/javaneturlequals-and-hashcode-make.html.
    try {
      if (lastUrl != null && url.toURI().equals(lastUrl.toURI())) {
        throw new HttpException("In re-direct loop");

      }
    } catch (URISyntaxException e) {
      // Do nothing, this is best effort.
    }
  }

  urlConnection = connectionFactory.build(url);
  for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
    urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
  }
  urlConnection.setConnectTimeout(timeout);
  urlConnection.setReadTimeout(timeout);
  urlConnection.setUseCaches(false);
  urlConnection.setDoInput(true);

  // Stop the urlConnection instance of HttpUrlConnection from following redirects so that
  // redirects will be handled by recursive calls to this method, loadDataWithRedirects.
  urlConnection.setInstanceFollowRedirects(false);

  // Connect explicitly to avoid errors in decoders if connection fails.
  urlConnection.connect();
  // Set the stream so that it's closed in cleanup to avoid resource leaks. See #2352.
  stream = urlConnection.getInputStream();
  if (isCancelled) {
    return null;
  }
  final int statusCode = urlConnection.getResponseCode();
  if (isHttpOk(statusCode)) {
    return getStreamForSuccessfulRequest(urlConnection);
  } else if (isHttpRedirect(statusCode)) {
    String redirectUrlString = urlConnection.getHeaderField("Location");
    if (TextUtils.isEmpty(redirectUrlString)) {
      throw new HttpException("Received empty or null redirect url");
    }
    URL redirectUrl = new URL(url, redirectUrlString);
    // Closing the stream specifically is required to avoid leaking ResponseBodys in addition
    // to disconnecting the url connection below. See #2352.
    cleanup();
    return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
  } else if (statusCode == INVALID_STATUS_CODE) {
    throw new HttpException(statusCode);
  } else {
    throw new HttpException(urlConnection.getResponseMessage(), statusCode);
  }
}
```

可以看到，Glide 加载图片默认是通过 HttpURLConnection 去加载的。

loadDataWithRedirects 方法建立链接返回一个读取的流 **InputStream。**获取到之后通过callback.onDataReady(result); 回调。而 callback 是 SourceGenerator 传进来的，所以看 SourceGenerator#onDataReady ：

```java
public void onDataReady(Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
        dataToCache = data;
        // We might be being called back on someone else's thread. Before doing anything, we should
        // reschedule to get back onto Glide's thread.
        cb.reschedule();
    } else {
        cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
                              loadData.fetcher.getDataSource(), originalKey);
    }
}
```

最后回调 cb 的 onDataFetcherReady，这里的 cb，是 DecodeJob：

```java
public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher,
                               DataSource dataSource, Key attemptedKey) {
    this.currentSourceKey = sourceKey;
    this.currentData = data;
    this.currentFetcher = fetcher;
    this.currentDataSource = dataSource;
    this.currentAttemptingKey = attemptedKey;
    if (Thread.currentThread() != currentThread) {
        runReason = RunReason.DECODE_DATA;
        callback.reschedule(this);
    } else {
        GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
        try {
            decodeFromRetrievedData();
        } finally {
            GlideTrace.endSection();
        }
    }
}
```

这个方法中，首先判断线程是不是当前线程，如果是的话调用 decodeFromRetrievedData。

```java
private void decodeFromRetrievedData() {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Retrieved data", startFetchTime,
                          "data: " + currentData
                          + ", cache key: " + currentSourceKey
                          + ", fetcher: " + currentFetcher);
    }
    Resource<R> resource = null;
    try {
        resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
        e.setLoggingDetails(currentAttemptingKey, currentDataSource);
        throwables.add(e);
    }
    if (resource != null) {
        notifyEncodeAndRelease(resource, currentDataSource);
    } else {
        runGenerators();
    }
}
```

创建 resource 对象，如果不为空执行下一步，空的话重新走 runGenerator 方法创建。

```java
private <Data> Resource<R> decodeFromData(DataFetcher<?> fetcher, Data data,
                                          DataSource dataSource) throws GlideException {
    try {
        if (data == null) {
            return null;
        }
        long startTime = LogTime.getLogTime();
        Resource<R> result = decodeFromFetcher(data, dataSource);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Decoded result " + result, startTime);
        }
        return result;
    } finally {
        fetcher.cleanup();
    }
}

private <Data> Resource<R> decodeFromFetcher(Data data, DataSource dataSource)
    throws GlideException {
    LoadPath<Data, ?, R> path = decodeHelper.getLoadPath((Class<Data>) data.getClass());
    return runLoadPath(data, dataSource, path);
}

<Data> LoadPath<Data, ?, Transcode> getLoadPath(Class<Data> dataClass) {
    return glideContext.getRegistry().getLoadPath(dataClass, resourceClass, transcodeClass);
}
```

decodeFormData 方法会调用 decodeFromFetcher 方法获取 Resource ，最后调用 HttpUrlFetcher 的 cleanup 关闭 stream 流和断开 HttpUrlConnection。

decodeFromFetcher 中直接调用了 Registry 的 getLoadPath 方法。注意到它的参数 dataClass 其实就是获取到传入的 InputStrem.class 对象。

```java
public <Data, TResource, Transcode> LoadPath<Data, TResource, Transcode> getLoadPath(
    @NonNull Class<Data> dataClass, @NonNull Class<TResource> resourceClass,
    @NonNull Class<Transcode> transcodeClass) {
    LoadPath<Data, TResource, Transcode> result =
        loadPathCache.get(dataClass, resourceClass, transcodeClass);
    if (loadPathCache.isEmptyLoadPath(result)) {
        return null;
    } else if (result == null) {
        List<DecodePath<Data, TResource, Transcode>> decodePaths =
            getDecodePaths(dataClass, resourceClass, transcodeClass);
        // It's possible there is no way to decode or transcode to the desired types from a given
        // data class.
        if (decodePaths.isEmpty()) {
            result = null;
        } else {
            result =
                new LoadPath<>(
                dataClass, resourceClass, transcodeClass, decodePaths, throwableListPool);
        }
        loadPathCache.put(dataClass, resourceClass, transcodeClass, result);
    }
    return result;
}
```

getLocalPath 方法第一个参数已经知道了，第二个参数 resourceClass 是 Object.class，第三个参数 transcodeClass 是 Drawable.class，这些变量都是初始化的时候创建好的。

首先先从缓存中获取 result ，如果获取不到，则创建一个，并加入缓存中。在创建 LoadPath 前，先创建一个DecodePath 对象，我们创建 Glide 的时候在 registry 中 append 的各种解析方式，getDecodePaths 就是根据我们传入的参数拿到对应的解析类。再然后创建出 LoadPath，传入刚创建的 DecodePath。

创建完 LoadPath 后，回到 decodeFromFetcher 方法中执行 runLoadPath。

```java
private <Data, ResourceType> Resource<R> runLoadPath(Data data, DataSource dataSource,
                                                     LoadPath<Data, ResourceType, R> path) throws GlideException {
    Options options = getOptionsWithHardwareConfig(dataSource);
    DataRewinder<Data> rewinder = glideContext.getRegistry().getRewinder(data);
    try {
        // ResourceType in DecodeCallback below is required for compilation to work with gradle.
        return path.load(
            rewinder, options, width, height, new DecodeCallback<ResourceType>(dataSource));
    } finally {
        rewinder.cleanup();
    }
}
```

path#load 方法调用的就是 LoadPath 的 load 方法：

```java
public Resource<Transcode> load(DataRewinder<Data> rewinder, @NonNull Options options, int width,
                                int height, DecodePath.DecodeCallback<ResourceType> decodeCallback) throws GlideException {
    List<Throwable> throwables = Preconditions.checkNotNull(listPool.acquire());
    try {
        return loadWithExceptionList(rewinder, options, width, height, decodeCallback, throwables);
    } finally {
        listPool.release(throwables);
    }
}

private Resource<Transcode> loadWithExceptionList(DataRewinder<Data> rewinder,
                                                  @NonNull Options options,
                                                  int width, int height, DecodePath.DecodeCallback<ResourceType> decodeCallback,
                                                  List<Throwable> exceptions) throws GlideException {
    Resource<Transcode> result = null;
    //noinspection ForLoopReplaceableByForEach to improve perf
    for (int i = 0, size = decodePaths.size(); i < size; i++) {
        DecodePath<Data, ResourceType, Transcode> path = decodePaths.get(i);
        try {
            result = path.decode(rewinder, width, height, options, decodeCallback);
        } catch (GlideException e) {
            exceptions.add(e);
        }
        if (result != null) {
            break;
        }
    }

    if (result == null) {
        throw new GlideException(failureMessage, new ArrayList<>(exceptions));
    }

    return result;
}
```

最后调用了 DecodePath 的 decode 进行解码并返回 Resource，在 decode 方法中，依次调用了 decodeResource->decodeResourceWithList 进行解码的实现。

```java
private Resource<ResourceType> decodeResourceWithList(DataRewinder<DataType> rewinder, int width,
                                                      int height, @NonNull Options options, List<Throwable> exceptions) throws GlideException {
    Resource<ResourceType> result = null;
    //noinspection ForLoopReplaceableByForEach to improve perf
    for (int i = 0, size = decoders.size(); i < size; i++) {
        ResourceDecoder<DataType, ResourceType> decoder = decoders.get(i);
        try {
            DataType data = rewinder.rewindAndGet();
            if (decoder.handles(data, options)) {
                data = rewinder.rewindAndGet();
                result = decoder.decode(data, width, height, options);
            }
            // Some decoders throw unexpectedly. If they do, we shouldn't fail the entire load path, but
            // instead log and continue. See #2406 for an example.
        } catch (IOException | RuntimeException | OutOfMemoryError e) {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "Failed to decode data for " + decoder, e);
            }
            exceptions.add(e);
        }

        if (result != null) {
            break;
        }
    }

    if (result == null) {
        throw new GlideException(failureMessage, new ArrayList<>(exceptions));
    }
    return result;
}
```

首先获取到 ResourceDecoder 解码器，然后通过 rewinder.rewindAndGet() 获取到 InputStream，然后调用 decode 方法解码，这里看 BitmapDrawableDecoder 的 decode 方法实现：


```java
public Resource<BitmapDrawable> decode(@NonNull DataType source, int width, int height,
                                       @NonNull Options options)
    throws IOException {
    Resource<Bitmap> bitmapResource = decoder.decode(source, width, height, options);
    return LazyBitmapDrawableResource.obtain(resources, bitmapResource);
}
```

BitmapDrawableDecoder 也是在创建 Glide 的时候实例化的，decoder 对象实现有三个：ByteBufferBitmapDecoder，StreamBitmapDecoder，还有一个叫 parcelFileDescriptorVideoDecoder 的对象。这里需要看的是 StreamBitmapDecoder：

```java
public Resource<Bitmap> decode(@NonNull InputStream source, int width, int height,
                               @NonNull Options options)
    throws IOException {
    final RecyclableBufferedInputStream bufferedStream;
    final boolean ownsBufferedStream;
    if (source instanceof RecyclableBufferedInputStream) {
        bufferedStream = (RecyclableBufferedInputStream) source;
        ownsBufferedStream = false;
    } else {
        bufferedStream = new RecyclableBufferedInputStream(source, byteArrayPool);
        ownsBufferedStream = true;
    }
    ExceptionCatchingInputStream exceptionStream =
        ExceptionCatchingInputStream.obtain(bufferedStream);
    MarkEnforcingInputStream invalidatingStream = new MarkEnforcingInputStream(exceptionStream);
    UntrustedCallbacks callbacks = new UntrustedCallbacks(bufferedStream, exceptionStream);
    try {
        return downsampler.decode(invalidatingStream, width, height, options, callbacks);
    } finally {
        exceptionStream.release();
        if (ownsBufferedStream) {
            bufferedStream.release();
        }
    }
}
```

RecyclableBufferedInputStream 是包装现有的 InputStream 并缓存输入的类。**而这里阿里有一个面试题是问到，为什么这里使用 ByteArrayPool ，为什么选择数组？**
 
然后通过 Downsampler#decode 方法去解码：


```java
public Resource<Bitmap> decode(InputStream is, int requestedWidth, int requestedHeight,
                               Options options, DecodeCallbacks callbacks) throws IOException {
    Preconditions.checkArgument(is.markSupported(), "You must provide an InputStream that supports"
                                + " mark()");

    byte[] bytesForOptions = byteArrayPool.get(ArrayPool.STANDARD_BUFFER_SIZE_BYTES, byte[].class);
    BitmapFactory.Options bitmapFactoryOptions = getDefaultOptions();
    bitmapFactoryOptions.inTempStorage = bytesForOptions;

    DecodeFormat decodeFormat = options.get(DECODE_FORMAT);
    DownsampleStrategy downsampleStrategy = options.get(DownsampleStrategy.OPTION);
    boolean fixBitmapToRequestedDimensions = options.get(FIX_BITMAP_SIZE_TO_REQUESTED_DIMENSIONS);
    boolean isHardwareConfigAllowed =
        options.get(ALLOW_HARDWARE_CONFIG) != null && options.get(ALLOW_HARDWARE_CONFIG);

    try {
        Bitmap result = decodeFromWrappedStreams(is, bitmapFactoryOptions,
                                                 downsampleStrategy, decodeFormat, isHardwareConfigAllowed, requestedWidth,
                                                 requestedHeight, fixBitmapToRequestedDimensions, callbacks);
        return BitmapResource.obtain(result, bitmapPool);
    } finally {
        releaseOptions(bitmapFactoryOptions);
        byteArrayPool.put(bytesForOptions);
    }
}
```

最终通过 decodeFromWrappedStreams 返回一个 Bitmap 对象，并把它保存到 BitmapResource 中返回。

回到一开始解码的地方，DecodeJob#decodeFromRetrievedData：

```java
private void decodeFromRetrievedData() {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Retrieved data", startFetchTime,
                          "data: " + currentData
                          + ", cache key: " + currentSourceKey
                          + ", fetcher: " + currentFetcher);
    }
    Resource<R> resource = null;
    try {
        resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
        e.setLoggingDetails(currentAttemptingKey, currentDataSource);
        throwables.add(e);
    }
    if (resource != null) {
        notifyEncodeAndRelease(resource, currentDataSource);
    } else {
        runGenerators();
    }
}
```

通过上面分析的一系列调用，获取到 resource 对象，如果不为 null ，就会调用 notifyEncodeAndRelease。

```java
private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
    if (resource instanceof Initializable) {
        ((Initializable) resource).initialize();
    }

    Resource<R> result = resource;
    LockedResource<R> lockedResource = null;
    if (deferredEncodeManager.hasResourceToEncode()) {
        lockedResource = LockedResource.obtain(resource);
        result = lockedResource;
    }

    notifyComplete(result, dataSource);

    stage = Stage.ENCODE;
    try {
        if (deferredEncodeManager.hasResourceToEncode()) {
            deferredEncodeManager.encode(diskCacheProvider, options);
        }
    } finally {
        if (lockedResource != null) {
            lockedResource.unlock();
        }
    }
    // Call onEncodeComplete outside the finally block so that it's not called if the encode process
    // throws.
    onEncodeComplete();
}
```

这个方法首先会调用 notifyComplete 通知解码完成，然后下面就开始编码，最后再调用 onEncodeComplete 方法通知编码完成。首先看 notifyComplete：

```java
private void notifyComplete(Resource<R> resource, DataSource dataSource) {
    setNotifiedOrThrow();
    callback.onResourceReady(resource, dataSource);
}
```

而这个 callback 是在 EngineJob 中实现的：

```java
public void onResourceReady(Resource<R> resource, DataSource dataSource) {
    synchronized (this) {
        this.resource = resource;
        this.dataSource = dataSource;
    }
    notifyCallbacksOfResult();
}

void notifyCallbacksOfResult() {
    ResourceCallbacksAndExecutors copy;
    Key localKey;
    EngineResource<?> localResource;
    synchronized (this) {
        stateVerifier.throwIfRecycled();
        if (isCancelled) {
            // TODO: Seems like we might as well put this in the memory cache instead of just recycling
            // it since we've gotten this far...
            resource.recycle();
            release();
            return;
        } else if (cbs.isEmpty()) {
            throw new IllegalStateException("Received a resource without any callbacks to notify");
        } else if (hasResource) {
            throw new IllegalStateException("Already have resource");
        }
        engineResource = engineResourceFactory.build(resource, isCacheable);
        // Hold on to resource for duration of our callbacks below so we don't recycle it in the
        // middle of notifying if it synchronously released by one of the callbacks. Acquire it under
        // a lock here so that any newly added callback that executes before the next locked section
        // below can't recycle the resource before we call the callbacks.
        hasResource = true;
        copy = cbs.copy();
        incrementPendingCallbacks(copy.size() + 1);

        localKey = key;
        localResource = engineResource;
    }

    listener.onEngineJobComplete(this, localKey, localResource);

    for (final ResourceCallbackAndExecutor entry : copy) {
        entry.executor.execute(new CallResourceReady(entry.cb));
    }
    decrementPendingCallbacks();
}
```

notifyCallbacksOfResult 中，看最后那部分，
**entry.executor.execute(new CallResourceReady(entry.cb))**;

entry.executor 是一个线程，在 ResourceCallbackAndExecutor 类里面封装，它在 copy 里面，cbs 是 ResourceCallbacksAndExecutors，copy 方法就是实例化自：


```java
ResourceCallbacksAndExecutors copy() {
    return new ResourceCallbacksAndExecutors(new ArrayList<>(callbacksAndExecutors));
}
```

而 callbacksAndExecutors 是通过 add 方法去赋值的：

```java
void add(ResourceCallback cb, Executor executor) {
    callbacksAndExecutors.add(new ResourceCallbackAndExecutor(cb, executor));
}
```

而 add 方法在 EngineJob#addCallback 中调用：

```java
synchronized void addCallback(final ResourceCallback cb, Executor callbackExecutor) {
    stateVerifier.throwIfRecycled();
    cbs.add(cb, callbackExecutor);
    //...
}
```

跟踪方法参数，发现 callbackExecutor 是在 into 方法中赋值的：

```java
public <Y extends Target<TranscodeType>> Y into(@NonNull Y target) {
    return into(target, /*targetListener=*/ null, Executors.mainThreadExecutor());
}

public final class Executors {
  private Executors() {
    // Utility class.
  }

  private static final Executor MAIN_THREAD_EXECUTOR =
      new Executor() {
        private final Handler handler = new Handler(Looper.getMainLooper());

        @Override
        public void execute(@NonNull Runnable command) {
          handler.post(command);
        }
      };

  /** Posts executions to the main thread. */
  public static Executor mainThreadExecutor() {
    return MAIN_THREAD_EXECUTOR;
  }
}
```

可以看到，**entry.executor.execute(new CallResourceReady(entry.cb))**;  其实就是把结果回调到主线程里面。

所以，这里可以直接看 CallResourceReady 的 run 方法：

```java
public void run() {
    synchronized (EngineJob.this) {
        if (cbs.contains(cb)) {
            // Acquire for this particular callback.
            engineResource.acquire();
            callCallbackOnResourceReady(cb);
            removeCallback(cb);
        }
        decrementPendingCallbacks();
    }
}

synchronized void callCallbackOnResourceReady(ResourceCallback cb) {
    try {
        // This is overly broad, some Glide code is actually called here, but it's much
        // simpler to encapsulate here than to do so at the actual call point in the
        // Request implementation.
        cb.onResourceReady(engineResource, dataSource);
    } catch (Throwable t) {
        throw new CallbackException(t);
    }
}
```

最后回调了 ResourceCallback#onResourceReady ，而 onResourceReady 的实现在 SingleRequest 里面。

```java
public synchronized void onResourceReady(Resource<?> resource, DataSource dataSource) {
    stateVerifier.throwIfRecycled();
    loadStatus = null;
    if (resource == null) {
        GlideException exception = new GlideException("Expected to receive a Resource<R> with an "
                                                      + "object of " + transcodeClass + " inside, but instead got null.");
        onLoadFailed(exception);
        return;
    }
    Object received = resource.get();
    if (received == null || !transcodeClass.isAssignableFrom(received.getClass())) {
        releaseResource(resource);
        GlideException exception = new GlideException("Expected to receive an object of "
                                                      + transcodeClass + " but instead" + " got "
                                                      + (received != null ? received.getClass() : "") + "{" + received + "} inside" + " "
                                                      + "Resource{" + resource + "}."
                                                      + (received != null ? "" : " " + "To indicate failure return a null Resource "
                                                         + "object, rather than a Resource object containing null data."));
        onLoadFailed(exception);
        return;
    }
    if (!canSetResource()) {
        releaseResource(resource);
        // We can't put the status to complete before asking canSetResource().
        status = Status.COMPLETE;
        return;
    }
    onResourceReady((Resource<R>) resource, (R) received, dataSource);
}
```

这里首先做一些失败的判断，并且调用 onLoadFailed 方法，最后调用 onResourceReady ：

```java
private synchronized void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
    // We must call isFirstReadyResource before setting status.
    boolean isFirstResource = isFirstReadyResource();
    status = Status.COMPLETE;
    this.resource = resource;
    if (glideContext.getLogLevel() <= Log.DEBUG) {
        Log.d(GLIDE_TAG, "Finished loading " + result.getClass().getSimpleName() + " from "
              + dataSource + " for " + model + " with size [" + width + "x" + height + "] in "
              + LogTime.getElapsedMillis(startTime) + " ms");
    }
    isCallingCallbacks = true;
    try {
        boolean anyListenerHandledUpdatingTarget = false;
        if (requestListeners != null) {
            for (RequestListener<R> listener : requestListeners) {
                anyListenerHandledUpdatingTarget |=
                    listener.onResourceReady(result, model, target, dataSource, isFirstResource);
            }
        }
        anyListenerHandledUpdatingTarget |=
            targetListener != null
            && targetListener.onResourceReady(result, model, target, dataSource, isFirstResource);
        
        if (!anyListenerHandledUpdatingTarget) {
            Transition<? super R> animation =
                animationFactory.build(dataSource, isFirstResource);
            target.onResourceReady(result, animation);
        }
    } finally {
        isCallingCallbacks = false;
    }
    notifyLoadSuccess();
}
```

这里有调用了 target.onResourceReady(result, animation); 我们最开始看的是 ImageViewTarget，所以这里进入 ImageViewTarget 的 onResourceReady 方法中：

```java
public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
    if (transition == null || !transition.transition(resource, this)) {
        setResourceInternal(resource);
    } else {
        maybeUpdateAnimatable(resource);
    }
}

private void setResourceInternal(@Nullable Z resource) {
    // Order matters here. Set the resource first to make sure that the Drawable has a valid and
    // non-null Callback before starting it.
    setResource(resource);
    maybeUpdateAnimatable(resource);
}
```

这里调用了 setResourceInternal 方法，最后调用 setResource 方法，setResource 方法是一个抽象方法。找到它的一个实现，在 BitmapImageViewTarget 里面：

```java
protected void setResource(Bitmap resource) {
    view.setImageBitmap(resource);
}
```

可以看到，直接把 bitmap 设置给 ImageView。至此，整个加载流程就结束了。
