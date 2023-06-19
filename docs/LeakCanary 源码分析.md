**基于 leakcanary-android:2.0-beta-3 版本**
 
**leakcanary2.0** 开始，只需要添加了依赖，就可以使用了，不需要再初始化：

![LeakCanary1.png](https://s2.loli.net/2023/06/19/tNqCDGX8PZWQm3r.png)

那么它是怎么做到的？

# 初始化
下载 LeakCanary 源码，发现有很多个 library。

![LeakCanary2.png](https://s2.loli.net/2023/06/19/6mKosfldb7gFWZt.png)

其中，在 **leakcanary-android-process **的清单文件中发现：

![LeakCanary3.png](https://s2.loli.net/2023/06/19/C6FpAyKOechHGMw.png)

LeakCanary 在另一个进程 leakcanary 中注册了一个 ContentProvider：

```kotlin
internal sealed class AppWatcherInstaller : ContentProvider() {

  internal class MainProcess : AppWatcherInstaller()

  internal class LeakCanaryProcess : AppWatcherInstaller() {
    override fun onCreate(): Boolean {
      super.onCreate()
      AppWatcher.config = AppWatcher.config.copy(enabled = false)
      return true
    }
  }

  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    InternalAppWatcher.install(application)
    return true
  }
  //... 
}
```

可以看到在 onCreate 中，使用了 InternalAppWatcher#install 方法进行初始化了，ContentProvider 和 Application 的执行顺序是：
**Application#attachBaseContext -> ContentProvider -> Application#onCreate**
所以，在 ContentProvider 中初始化是没问题的。

# InternalAppWatcher#install

```kotlin
fun install(application: Application) {
  SharkLog.logger = DefaultCanaryLog()
  SharkLog.d { "Installing AppWatcher" }
  //检查是否是主线程
  checkMainThread() 
  if (this::application.isInitialized) {
    return
  }
  //保存 application
  InternalAppWatcher.application = application
  //获取配置
  val configProvider = { AppWatcher.config }
  //监听 Activity onDestroy 
  ActivityDestroyWatcher.install(application, objectWatcher, configProvider)
  //监听 Fragment onDestroy
  FragmentDestroyWatcher.install(application, objectWatcher, configProvider)
  onAppWatcherInstalled(application)
}
```

install 方法里面监听了 Activity 和 Fragment 的 **onDestroy** 方法，然后回调到 **objectWatcher** 里面。

# ActivityDestroyWatcher

```kotlin
internal class ActivityDestroyWatcher private constructor(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        if (configProvider().watchActivities) {
          objectWatcher.watch(activity)
        }
      }
    }

  companion object {
    fun install(
      application: Application,
      objectWatcher: ObjectWatcher,
      configProvider: () -> Config
    ) {
      val activityDestroyWatcher =
        ActivityDestroyWatcher(objectWatcher, configProvider)
      application.registerActivityLifecycleCallbacks(activityDestroyWatcher.lifecycleCallbacks)
    }
  }
}
```

可以看到，在 install 方法中，监听 Activity 的方法是通过用 application 注册了 registerActivityLifecycleCallbacks 回调监听的。只监听了 **onActivityDestroyed** 回调，configProvider 是传进来的配置信息，可控制是否监听。最后通过 **objectWatcher#watch** 方法回调出去。

# FragmentDestroyWatcher

```kotlin
internal object FragmentDestroyWatcher {

  private const val ANDROIDX_FRAGMENT_CLASS_NAME = "androidx.fragment.app.Fragment"
  private const val ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME =
    "leakcanary.internal.AndroidXFragmentDestroyWatcher"

  fun install(
    application: Application,
    objectWatcher: ObjectWatcher,
    configProvider: () -> AppWatcher.Config
  ) {
    val fragmentDestroyWatchers = mutableListOf<(Activity) -> Unit>()
    //大于android O 时添加 Fragment 监听
    if (SDK_INT >= O) {
      fragmentDestroyWatchers.add(
          AndroidOFragmentDestroyWatcher(objectWatcher, configProvider)
      )
    }
	//添加 AndroidX 的 Fragment 监听
    if (classAvailable(ANDROIDX_FRAGMENT_CLASS_NAME) &&
        classAvailable(ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME)
    ) {
      val watcherConstructor = Class.forName(ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME)
          .getDeclaredConstructor(ObjectWatcher::class.java, Function0::class.java)
      @kotlin.Suppress("UNCHECKED_CAST")
      fragmentDestroyWatchers.add(
          watcherConstructor.newInstance(objectWatcher, configProvider) as (Activity) -> Unit
      )
    }
	//如果没添加到，就直接返回结束
    if (fragmentDestroyWatchers.size == 0) {
      return
    }
	//监听
    application.registerActivityLifecycleCallbacks(object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityCreated(
        activity: Activity,
        savedInstanceState: Bundle?
      ) {
        for (watcher in fragmentDestroyWatchers) {
          watcher(activity)
        }
      }
    })
  }

  private fun classAvailable(className: String): Boolean {
    return try {
      Class.forName(className)
      true
    } catch (e: ClassNotFoundException) {
      false
    }
  }
}
```

首先通过判断，添加 android O 及以上版本的 Fragment 以及 AndroidX 的 Fragment 监听支持。所以 **LeakCanary 是不支持 Android O 以下的 Fragment 监听的。**
**
## android O Fragment 监听

```kotlin
internal class AndroidOFragmentDestroyWatcher(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) : (Activity) -> Unit {
  private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {

    override fun onFragmentViewDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      val view = fragment.view
      if (view != null && configProvider().watchFragmentViews) {
        objectWatcher.watch(view)
      }
    }

    override fun onFragmentDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      if (configProvider().watchFragments) {
        objectWatcher.watch(fragment)
      }
    }
  }

  override fun invoke(activity: Activity) {
    val fragmentManager = activity.fragmentManager
    fragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
  }
}
```

通过 FragmentManager 注册 FragmentLifecycleCallbacks 监听，监听了 **onFragmentViewDestroyed** 以及 **onFragmentDestroyed** 回调。它们分别对应 onDestroyView 和  onDestroy 方法。

## androidX Fragment 监听
androidX 监听支持实现在 AndroidXFragmentDestroyWatcher 类中。

```kotlin
internal class AndroidXFragmentDestroyWatcher(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) : (Activity) -> Unit {

  private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {

    override fun onFragmentViewDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      val view = fragment.view
      if (view != null && configProvider().watchFragmentViews) {
        objectWatcher.watch(view)
      }
    }

    override fun onFragmentDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      if (configProvider().watchFragments) {
        objectWatcher.watch(fragment)
      }
    }
  }

  override fun invoke(activity: Activity) {
    if (activity is FragmentActivity) {
      val supportFragmentManager = activity.supportFragmentManager
      supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
    }
  }
}
```

原理一样。可以留意到这两个类都继承了：(Activity) -> Unit 表示接收类型为 Activity 参数并返回不指定类型。这跟 invoke 方法的参数有关。

最后

```kotlin
application.registerActivityLifecycleCallbacks(object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
    override fun onActivityCreated(
        activity: Activity,
        savedInstanceState: Bundle?
            ) {
        for (watcher in fragmentDestroyWatchers) {
            watcher(activity)
        }
    }
})
```

通过监听 Activity 的 onCreate 方法调用 Fragment 监听类里面的 invoke 方法注册监听器监听。

# ObjectWatcher#watch

```kotlin
@Synchronized fun watch(watchedObject: Any) {
    watch(watchedObject, "")
}

@Synchronized fun watch(
    watchedObject: Any,
    name: String
) {
    if (!isEnabled()) {
        return
    }
    removeWeaklyReachableObjects()
    val key = UUID.randomUUID()
        .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    val reference =
        KeyedWeakReference(watchedObject, key, name, watchUptimeMillis, queue)
    SharkLog.d {
        "Watching " +
            (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
            (if (name.isNotEmpty()) " named $name" else "") +
            " with key $key"
    }

    watchedObjects[key] = reference
    checkRetainedExecutor.execute {
        moveToRetained(key)
    }
}
```

首先，调用 removeWeaklyReachableObjects 方法移除队列中将要被 GC 的引用。

```kotlin
private fun removeWeaklyReachableObjects() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    var ref: KeyedWeakReference?
        do {
            ref = queue.poll() as KeyedWeakReference?
                if (ref != null) {
                    watchedObjects.remove(ref.key)
                }
        } while (ref != null)
}
```

这里有几个知识点：

```kotlin
private val watchedObjects = mutableMapOf<String, KeyedWeakReference>()

private val queue = ReferenceQueue<Any>()

class KeyedWeakReference(
  referent: Any,
  val key: String,
  val name: String,
  val watchUptimeMillis: Long,
  referenceQueue: ReferenceQueue<Any>
) : WeakReference<Any>(
    referent, referenceQueue
) {
  @Volatile
  var retainedUptimeMillis = -1L

  companion object {
    @Volatile
    @JvmStatic var heapDumpUptimeMillis = 0L
  }
}
```

1. watchedObjects ：**watch() 方法传进来的引用，尚未判定为泄露。**
2. queue：** 引用队列，配合弱引用使用**
3. KeyedWeakReference ：**弱引用**

通过 watch 方法传进来的引用，即 watchedObject ，都会通过：

**val reference = KeyedWeakReference(watchedObject, key, name, watchUptimeMillis, queue)**
**
保存到 KeyedWeakReference 中，key 为 UUID，是唯一的，**queue 在这此时和弱引用关联**。reference 会通过 key 放在 watchedObjects 中。

queue 是一个 ReferenceQueue 队列，配合弱引用使用。**弱引用一旦变得弱可达，就会立即入队。这将在 finalization 或者 GC 之前发生。**也就是说，**会被 GC 回收的对象引用，会保存在队列 queue 中。**

在 removeWeaklyReachableObjects 方法中，循环从 queue 队列中取出**被 GC 回收的对象引用，判断如果不为空，则将这个引用在 watchedObjects 中移除。循环结束后，如果还有对象没有移除，那么剩下的就暂时标记为泄露的对象。**
**
```kotlin
checkRetainedExecutor.execute {
    moveToRetained(key)
}
//...
private val mainHandler = Handler(Looper.getMainLooper())

private val checkRetainedExecutor = Executor {
    mainHandler.postDelayed(it, AppWatcher.config.watchDurationMillis)
}

val objectWatcher = ObjectWatcher(
    clock = clock,
    checkRetainedExecutor = checkRetainedExecutor,
    isEnabled = { AppWatcher.config.enabled }
)
```

checkRetainedExecutor 是线程接口，它在这里的实现是 Handler 延时 post 一个 Runnable。默认时间是 5 秒。

```kotlin
@Synchronized private fun moveToRetained(key: String) {
    removeWeaklyReachableObjects()
    val retainedRef = watchedObjects[key]
    if (retainedRef != null) {
        retainedRef.retainedUptimeMillis = clock.uptimeMillis()
        onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
}
```

在 moveToRetained 中，首先再次调用 removeWeaklyReachableObjects 方法，防止遗漏。然后 在watchedObjects 中获取泄的对象 retainedRef。给 retainedUptimeMillis 属性赋值，用于后面计算个数用。最后回调 OnObjectRetainedListener 的 onObjectRetained 方法。

OnObjectRetainedListener 是通过 addOnObjectRetainedListener 方法添加监听的。而 addOnObjectRetainedListener 方法是在 InternalLeakCanary 类中调用的。而 InternalLeakCanary 又是在 InternalAppWatcher 初始化时初始化的：

```kotlin
init {
    val internalLeakCanary = try {
        val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
        leakCanaryListener.getDeclaredField("INSTANCE")
            .get(null)
    } catch (ignored: Throwable) {
        NoLeakCanary
    }
    @kotlin.Suppress("UNCHECKED_CAST")
    onAppWatcherInstalled = internalLeakCanary as (Application) -> Unit
}
```

通过反射调用了 InternalLeakCanary 的 invike 方法：

```kotlin
//InternalLeakCanary#invoke
override fun invoke(application: Application) {
    this.application = application
    AppWatcher.objectWatcher.addOnObjectRetainedListener(this)
    //...
}
```

# OnObjectRetainedListener#onObjectRetained
接下来看看 onObjectRetained 回调方法：

```kotlin
override fun onObjectRetained() {
    if (this::heapDumpTrigger.isInitialized) {
        heapDumpTrigger.onObjectRetained()
    }
}

fun onObjectRetained() {
    scheduleRetainedObjectCheck("found new object retained")
}

private fun scheduleRetainedObjectCheck(reason: String) {
    if (checkScheduled) {
        SharkLog.d { "Already scheduled retained check, ignoring ($reason)" }
        return
    }
    checkScheduled = true
    backgroundHandler.post {
        checkScheduled = false
        checkRetainedObjects(reason)
    }
}
```

onObjectRetained 回调方法最终调用了 HeapDumpTrigger 的 scheduleRetainedObjectCheck 方法。在 scheduleRetainedObjectCheck 方法中，在子线程执行了 **checkRetainedObjects** 方法。

**checkRetainedObjects** 方法是确定泄露的最后一个方法了。这里会确认引用是否真的泄露，如果真的泄露，则发起 heap dump，分析 dump 文件，找到引用链，最后通知用户。整体流程和老版本是一致的，但在一些细节处理，以及 dump 文件的分析上有所区别。下面还是通过源码来看看这些区别。

# checkRetainedObjects

```kotlin
private fun checkRetainedObjects(reason: String) {
    val config = configProvider()
    // A tick will be rescheduled when this is turned back on.
    if (!config.dumpHeap) {
        SharkLog.d { "No checking for retained object: LeakCanary.Config.dumpHeap is false" }
        return
    }
    SharkLog.d { "Checking retained object because $reason" }
	//获取在 ObjectWatcher 中计算出来的内存泄露实例个数
    var retainedReferenceCount = objectWatcher.retainedObjectCount
	//如果大于0,则手动执行一次 GC 操作
    if (retainedReferenceCount > 0) {
        gcTrigger.runGc()
        //再获取一次
        retainedReferenceCount = objectWatcher.retainedObjectCount
    }
	//如果泄露实例个数小于 5 个，不进行 heap dump
    if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return
	
    if (!config.dumpHeapWhenDebugging && DebuggerControl.isDebuggerAttached) {
        showRetainedCountWithDebuggerAttached(retainedReferenceCount)
        scheduleRetainedObjectCheck("debugger was attached", WAIT_FOR_DEBUG_MILLIS)
        SharkLog.d {
            "Not checking for leaks while the debugger is attached, will retry in $WAIT_FOR_DEBUG_MILLIS ms"
        }
        return
    }

    SharkLog.d { "Found $retainedReferenceCount retained references, dumping the heap" }
    val heapDumpUptimeMillis = SystemClock.uptimeMillis()
    KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
    dismissRetainedCountNotification()
    //AndroidHeapDumper
    val heapDumpFile = heapDumper.dumpHeap()
    if (heapDumpFile == null) {
        SharkLog.d { "Failed to dump heap, will retry in $WAIT_AFTER_DUMP_FAILED_MILLIS ms" }
        scheduleRetainedObjectCheck("failed to dump heap", WAIT_AFTER_DUMP_FAILED_MILLIS)
        showRetainedCountWithHeapDumpFailed(retainedReferenceCount)
        return
    }
    lastDisplayedRetainedObjectCount = 0
    //移除已经 heap dump 的 retainedKeys
    objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)
    // 分析 heap dump 文件
    HeapAnalyzerService.runAnalysis(application, heapDumpFile)
}
```

 首先获取内存泄露实例个数 retainedReferenceCount，它的获取实现是遍历 ObjectWatcher 中的 watchedObjects 列表，获取里面 retainedUptimeMillis 不等于 -1 的个数，这个属性的赋值上面有说。

```kotlin
val retainedObjectCount: Int
@Synchronized get() {
    removeWeaklyReachableObjects() //执行一次，以免遗漏
    return watchedObjects.count { it.value.retainedUptimeMillis != -1L }
}
```

接下来手动执行一次 GC，完了之后再次获取内存泄露实例个数：

```kotlin
if (retainedReferenceCount > 0) {
    gcTrigger.runGc()
    retainedReferenceCount = objectWatcher.retainedObjectCount
}

interface GcTrigger {
  fun runGc()
  
  object Default : GcTrigger {
    //手动GC
    override fun runGc() {
      Runtime.getRuntime().gc()
      enqueueReferences()
      System.runFinalization()
    }

    private fun enqueueReferences() {
      try {
        Thread.sleep(100)
      } catch (e: InterruptedException) {
        throw AssertionError()
      }
    }
  }
}
```

然后调用 checkRetainedCount 方法检查泄露实例个数是否小于 5 个，如果小于就不进行 heap dump，直接 return。只是弹出一个通知提示，并在 5 秒后再次调用 checkRetainedObjects 方法检测。config.retainedVisibleThreshold 的默认大小是 5。
在 checkRetainedCount 方法中最后是通过调用 scheduleRetainedObjectCheck 方法实现 5 秒后再次检测功能的，原理是 Handler。代码就不贴了。

接下来的逻辑就是创建 dump heap 文件： **val heapDumpFile = heapDumper.dumpHeap()。**
HeapDumper 是一个接口，它的实现类是 AndroidHeapDumper，dumpHeap 方法里面就是创建文件的一些 IO 操作。
然后通过 **objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)  **移除已经 heap dump 的 retainedKeys。
最后通过 **HeapAnalyzerService#runAnalysis** 方法分析 heap dump 文件。

在 dumpHeap 方法中，核心方法是：**Debug.dumpHprofData(heapDumpFile._absolutePath_)**
**
# runAnalysis 

```kotlin
companion object {
    private const val HEAPDUMP_FILE_EXTRA = "HEAPDUMP_FILE_EXTRA"

    fun runAnalysis(
        context: Context,
        heapDumpFile: File
    ) {
        val intent = Intent(context, HeapAnalyzerService::class.java)
        intent.putExtra(HEAPDUMP_FILE_EXTRA, heapDumpFile)
        startForegroundService(context, intent)
    }

    fun startForegroundService(
        context: Context,
        intent: Intent
    ) {
        if (SDK_INT >= 26) {
            context.startForegroundService(intent)
        } else {
            // Pre-O behavior.
            context.startService(intent)
        }
    }
}
```

在 runAnalysis 方法中，可以看到它是启动一个前台服务 HeapAnalyzerService 来分析 heap dump 文件的。具体是如何分析的，是通过它的自己写的 shark 包中实现的，旧版本用的是 haha 库，但已经弃用了。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/450005/1571987649236-b7fed8cb-56b7-4010-bfc1-39ba3393384d.png#align=left&display=inline&height=255&originHeight=142&originWidth=366&size=35172&status=done&width=657)

关于 Shark 库的介绍可以看这里：[https://square.github.io/leakcanary/shark/](https://square.github.io/leakcanary/shark/)
主要有点是快速和低内存。
