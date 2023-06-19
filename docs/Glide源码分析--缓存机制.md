# 内存缓存

在上一篇中分析了 load 和 into 加载图片的一个流程。现在我们重新看看 Engine#load 方法：

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
    //...
  }
```

这次看这个方法的前半部分，是关于 Glide 加载图片时缓存的部分。在内存缓存方面，Glide 使用了两种方式，一种是 WeakReference 弱引用方式，一种是 LruCache 算法。

在 load 方法中：

1. 首先根据很多参数创建了一个 EngineKey 类型的 key，这个 key 就是下面缓存存取时使用的 key。之所以使用这么多参数创建，也是为了这个 key 的准确性，比如同一张图片我们给它的长宽不一样，所生成的 key 就不一样认为是两张图片。 

2. 接下来是先从弱引用中取图片，active 是活动的意思，**其实弱引用中缓存的图片是我们现在正在使用的图片，之所以多出来这个弱引用缓存主要为了当前使用的图片过多的时候可以防止这些图片被 LruCache 算法回收掉**  ：

```java
EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);

private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
        return null;
    }
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
        active.acquire();
    }
    return active;
}
```

activeResources 是 ActiveResources 对象，ActiveResources#get：

```java
synchronized EngineResource<?> get(Key key) {
    ResourceWeakReference activeRef = activeEngineResources.get(key);
    if (activeRef == null) {
        return null;
    }

    EngineResource<?> active = activeRef.get();
    if (active == null) {
        cleanupActiveReference(activeRef);
    }
    return active;
}
```

activeEngineResources 是一个 HashMap：


```java
 final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();

static final class ResourceWeakReference extends WeakReference<EngineResource<?>> {
    @SuppressWarnings("WeakerAccess") @Synthetic final Key key;
    @SuppressWarnings("WeakerAccess") @Synthetic final boolean isCacheable;

    @Nullable @SuppressWarnings("WeakerAccess") @Synthetic Resource<?> resource;

    @Synthetic
    @SuppressWarnings("WeakerAccess")
    ResourceWeakReference(
        @NonNull Key key,
        @NonNull EngineResource<?> referent,
        @NonNull ReferenceQueue<? super EngineResource<?>> queue,
        boolean isActiveResourceRetentionAllowed) {
        super(referent, queue);
        this.key = Preconditions.checkNotNull(key);
        this.resource =
            referent.isCacheable() && isActiveResourceRetentionAllowed
            ? Preconditions.checkNotNull(referent.getResource()) : null;
        isCacheable = referent.isCacheable();
    }

    void reset() {
        resource = null;
        clear();
    }
}
```

HashMap 内部存的就是我们的弱引用对象。弱引用中保存了我们的资源对象 EngineResource。 

3. 如果弱引用中取不到，则到 LruCache 中查找。
```java
 EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);

private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
        return null;
    }
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
        cached.acquire();
        activeResources.activate(key, cached);
    }
    return cached;
}

private EngineResource<?> getEngineResourceFromCache(Key key) {
    Resource<?> cached = cache.remove(key);
    final EngineResource<?> result;
    if (cached == null) {
        result = null;
    } else if (cached instanceof EngineResource) {
        // Save an object allocation if we've cached an EngineResource (the typical case).
        result = (EngineResource<?>) cached;
    } else {
        result = new EngineResource<>(cached, true /*isMemoryCacheable*/, true /*isRecyclable*/);
    }
}
```

其中，cache 对象是 MemoryCache 接口，它有两个实现类，在 GlideBuilder 中创建 Engine 对象时，传入的实现类是 LruResourceCache。而 LruResourceCache 继承了 LruCache，所以我们知道这是在 LruCache 中取资源。

在 getEngineResourceFromCache 方法中，取缓存是使用
 `Resource<?> cached = cache.remove(key); ` 去取的，所以是取完缓存就把缓存从 LruCache 中删除并返回。取完回到 loadFromCache 方法中，如果缓存不为空的话，就会调用 `activeResources.activate(key, cached);`  把缓存存在弱引用中。

**内存缓存是何时存进去的？**
 
在上一篇分析中，可以知道 Glide 在加载完后会回调 EngineJonb 的 onResourceReady 方，在 onResourceReady 里面会调用 notifyCallbacksOfException：

```java
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
 
其中**listener.onEngineJobComplete(this, localKey, localResource);** 回调的实现在 Engine 里面：

```java
public synchronized void onEngineJobComplete(
    EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
    // A null resource indicates that the load failed, usually due to an exception.
    if (resource != null) {
        resource.setResourceListener(key, this);

        if (resource.isCacheable()) {
            activeResources.activate(key, resource);
        }
    }

    jobs.removeIfCurrent(key, engineJob);
}
```

activeResources 就是上面分析过的弱引用，所以在这个回调里，是把当前的 resource 保存在弱引用中。

再看 notifyCallbacksOfException 最后调用了 decrementPendingCallbacks 方法：

```java
synchronized void decrementPendingCallbacks() {
    stateVerifier.throwIfRecycled();
    Preconditions.checkArgument(isDone(), "Not yet complete!");
    int decremented = pendingCallbacks.decrementAndGet();
    Preconditions.checkArgument(decremented >= 0, "Can't decrement below 0");
    if (decremented == 0) {
        if (engineResource != null) {
            engineResource.release();
        }

        release();
    }
}
```

在 decrementPendingCallbacks 中，调用了 EngineResource 的 release。

```java
void release() {
    // To avoid deadlock, always acquire the listener lock before our lock so that the locking
    // scheme is consistent (Engine -> EngineResource). Violating this order leads to deadlock
    // (b/123646037).
    synchronized (listener) {
        synchronized (this) {
            if (acquired <= 0) {
                throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
            }
            if (--acquired == 0) {
                listener.onResourceReleased(key, this);
            }
        }
    }
}
```

然后回调了 onResourceReleased，onResourceReleased 方法的实现也在 Engine 中：

```java
public synchronized void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    activeResources.deactivate(cacheKey);
    if (resource.isCacheable()) {
        cache.put(cacheKey, resource);
    } else {
        resourceRecycler.recycle(resource);
    }
}
```

所以这里看到了把 resource 保存在了 Lru 中。

在 notifyCallbacksOfResult 方法中还看到了**在弱引用之前**调用了 incrementPendingCallbacks 方法：


```java
synchronized void incrementPendingCallbacks(int count) {
    Preconditions.checkArgument(isDone(), "Not yet complete!");
    if (pendingCallbacks.getAndAdd(count) == 0 && engineResource != null) {
        engineResource.acquire();
    }
}

synchronized void acquire() {
    if (isRecycled) {
        throw new IllegalStateException("Cannot acquire a recycled resource");
    }
    ++acquired;
}
```

在 incrementPendingCallbacks 方法里最终调用里调用了 engineResource.acquire();   
然方法最后在  decrementPendingCallbacks 中又调用了 engineResource.release() ;   
engineResource 是我们的图片资源的包装对象。  
acquire()  和 release() 方法都是对齐成员变量。  
acquired 的操控 acquire() 是给 acquired 加一。  
release() 是给 acquired 减一。  
**当 acquired 大于 0 的时候说明这个图片我们正在使用，当他等于 0 的时候，说明现在没使用可以存在 LruCache中。**
 
# 硬盘缓存


硬盘缓存是耗时操作，在子线程中执行，所以，这里直接看 Engine#load 方法的加载部分，DecodeJob 的 run 方法。在 run 方法中，调用了 runWrapped：

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
```
 
这里判断的 runReason 是这样的：  
**INITIALIZE：第一次加载的时候**  
**SWITCH_TO_SOURCE_SERVICE：从硬盘缓存中取**  
**DECODE_DATA：解析从网络加载完的数据**

如果是在磁盘里面取缓存的话， currentGenerator 获取到的是 ResourceCacheGenerator，接下来的 runGenerators 中，执行的是 startNext 方法：


```java
public boolean startNext() {
    List<Key> sourceIds = helper.getCacheKeys();
    if (sourceIds.isEmpty()) {
        return false;
    }
    List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();
    if (resourceClasses.isEmpty()) {
        if (File.class.equals(helper.getTranscodeClass())) {
            return false;
        }
        throw new IllegalStateException(
            "Failed to find any load path from " + helper.getModelClass() + " to "
            + helper.getTranscodeClass());
    }
    while (modelLoaders == null || !hasNextModelLoader()) {
        resourceClassIndex++;
        if (resourceClassIndex >= resourceClasses.size()) {
            sourceIdIndex++;
            if (sourceIdIndex >= sourceIds.size()) {
                return false;
            }
            resourceClassIndex = 0;
        }

        Key sourceId = sourceIds.get(sourceIdIndex);
        Class<?> resourceClass = resourceClasses.get(resourceClassIndex);
        Transformation<?> transformation = helper.getTransformation(resourceClass);
        // PMD.AvoidInstantiatingObjectsInLoops Each iteration is comparatively expensive anyway,
        // we only run until the first one succeeds, the loop runs for only a limited
        // number of iterations on the order of 10-20 in the worst case.
        currentKey =
            new ResourceCacheKey(// NOPMD AvoidInstantiatingObjectsInLoops
            helper.getArrayPool(),
            sourceId,
            helper.getSignature(),
            helper.getWidth(),
            helper.getHeight(),
            transformation,
            resourceClass,
            helper.getOptions());
        cacheFile = helper.getDiskCache().get(currentKey);
        if (cacheFile != null) {
            sourceKey = sourceId;
            modelLoaders = helper.getModelLoaders(cacheFile);
            modelLoaderIndex = 0;
        }
    }

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
        ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
        loadData = modelLoader.buildLoadData(cacheFile,
                                             helper.getWidth(), helper.getHeight(), helper.getOptions());
        if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
            started = true;
            loadData.fetcher.loadData(helper.getPriority(), this);
        }
    }

    return started;
}
```

41 行看到 **cacheFile = helper.getDiskCache().get(currentKey); ，**

getDiskCache 是获取 DiskCache 接口，它的实现类是 DiskLruCacheWrapper，get 方法：

```java
public File get(Key key) {
    String safeKey = safeKeyGenerator.getSafeKey(key);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Get: Obtained: " + safeKey + " for for Key: " + key);
    }
    File result = null;
    try {
        // It is possible that the there will be a put in between these two gets. If so that shouldn't
        // be a problem because we will always put the same value at the same key so our input streams
        // will still represent the same data.
        final DiskLruCache.Value value = getDiskCache().get(safeKey);
        if (value != null) {
            result = value.getFile(0);
        }
    } catch (IOException e) {
        if (Log.isLoggable(TAG, Log.WARN)) {
            Log.w(TAG, "Unable to get from disk cache", e);
        }
    }
    return result;
}
```

在 get 方法中，可以看到是在 DiskLruCache 中取的缓存。

**那么磁盘缓存是在哪里存进去的呢？**
 
存磁盘缓存一般都是加载完图片后再存储的，在上一篇的分析中知道，Glide 加载完后会回调 callback 的 onDataReady 方法，onDataReady 是在 SourceGenerator 中实现的：

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

DiskCacheStrategy 是缓存策略，缓存策略我们从开始初始化的时候可以配置：

```java
 RequestOptions options = new RequestOptions()
     .skipMemoryCache(true)//跳过内存缓存
     .diskCacheStrategy(DiskCacheStrategy.ALL)//缓存所有版本的图像
     .diskCacheStrategy(DiskCacheStrategy.NONE)//跳过磁盘缓存
     .diskCacheStrategy(DiskCacheStrategy.DATA)//只缓存原来分辨率的图片
     .diskCacheStrategy(DiskCacheStrategy.RESOURCE)//只缓存最终的图片
```

在 onDataReady 中，如果允许缓存就给 dataToCache 赋值，然后回调 reschedule，这里 cb 就是 DecodeJob：

```java
public void reschedule() {
    runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
    callback.reschedule(this);
}
```

callback.reschedule 的实现则在 EngineJob 里面 ：


```java
public void reschedule(DecodeJob<?> job) {
    // Even if the job is cancelled here, it still needs to be scheduled so that it can clean itself
    // up.
    getActiveSourceExecutor().execute(job);
}
```

可以看到更改 runReason 为 SWITCH_TO_SOURCE_SERVICE 然后又执行线程里的工作，所以又会回到 run() 方法中执行 runGenerators()。在 runGenerators 中再次执行 startNext 方法，这次 currentGenerator 的实现变成了 SourceGenerator。

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

因为 dataToCache 上上面的时候已经赋值了，所以不为空，所以进入了 cacheData 方法里面：

```java
private void cacheData(Object dataToCache) {
    long startTime = LogTime.getLogTime();
    try {
        Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
        DataCacheWriter<Object> writer =
            new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
        originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
        helper.getDiskCache().put(originalKey, writer);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Finished encoding source to cache"
                  + ", key: " + originalKey
                  + ", data: " + dataToCache
                  + ", encoder: " + encoder
                  + ", duration: " + LogTime.getElapsedMillis(startTime));
        }
    } finally {
        loadData.fetcher.cleanup();
    }

    sourceCacheGenerator =
        new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
}
```

可以看到通过 **helper.getDiskCache().put(originalKey, writer);**把缓存存在浏览 DiskCache 中。
