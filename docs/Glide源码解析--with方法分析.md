![glide.png](https://s2.loli.net/2023/06/19/vA7S4xQswdJWKX8.png)

**Glide 目录结构**

Glide 可以大概分为下面几部分：
加载请求，执行引擎，数据加载，解码器，编码器，缓存...

Glide 的简单使用：

```
 Glide.with(this).load("").into(imageView);
```

第一次使用 Glide 的加载流程：

![glide1.png](https://s2.loli.net/2023/06/19/uf87XbBGPTDl2wz.png)

**Glide#with 方法**

![glide2.png](https://s2.loli.net/2023/06/19/BApmJyrFDTYKVMU.png)

with 方法有 6 个重载，他们都返回  RequestManager 对象，他们的实现都是通过 getRetriever 方法获取到 RequestManagerRetriever 对象，然后再通过 RequestManagerRetriever#get 去获取的。

```java
  public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
  }

  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // Context could be null for other reasons (ie the user passes in null), but in practice it will
    // only occur due to errors with the Fragment lifecycle.
    Preconditions.checkNotNull(
        context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
  }
```

这里看 Glide#get 方法：

```java
@NonNull
public static Glide get(@NonNull Context context) {
    if (glide == null) {
        synchronized (Glide.class) {
            if (glide == null) {
                checkAndInitializeGlide(context);
            }
        }
    }
    return glide;
}

private static void checkAndInitializeGlide(@NonNull Context context) {
    // In the thread running initGlide(), one or more classes may call Glide.get(context).
    // Without this check, those calls could trigger infinite recursion.
    if (isInitializing) {
        throw new IllegalStateException("You cannot call Glide.get() in registerComponents(),"
                                        + " use the provided Glide instance instead");
    }
    isInitializing = true;
    initializeGlide(context);
    isInitializing = false;
}
```

**很明显，Glide 对象的创建是一个单例模式。并且是在 initializeGlide 方法中创建的。**

```java
private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
    Context applicationContext = context.getApplicationContext();
    GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
    List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
    if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
      manifestModules = new ManifestParser(applicationContext).parse();
    }

    if (annotationGeneratedModule != null
        && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
      Set<Class<?>> excludedModuleClasses =
          annotationGeneratedModule.getExcludedModuleClasses();
      Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
      while (iterator.hasNext()) {
        com.bumptech.glide.module.GlideModule current = iterator.next();
        if (!excludedModuleClasses.contains(current.getClass())) {
          continue;
        }
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "AppGlideModule excludes manifest GlideModule: " + current);
        }
        iterator.remove();
      }
    }

    if (Log.isLoggable(TAG, Log.DEBUG)) {
      for (com.bumptech.glide.module.GlideModule glideModule : manifestModules) {
        Log.d(TAG, "Discovered GlideModule from manifest: " + glideModule.getClass());
      }
    }

    RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
            ? annotationGeneratedModule.getRequestManagerFactory() : null;
    builder.setRequestManagerFactory(factory);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
      module.applyOptions(applicationContext, builder);
    }
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.applyOptions(applicationContext, builder);
    }
    Glide glide = builder.build(applicationContext);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
      module.registerComponents(applicationContext, glide, glide.registry);
    }
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
    }
    applicationContext.registerComponentCallbacks(glide);
    Glide.glide = glide;
  }
```

通过 initializeGlide 的源码发现，首先，它会查找被 GlideModule 注解标记的类，如果找不到，即 annotationGeneratedModule==null 的话，会继续遍历 AndroidManifest.xml 文件去查找，并赋值给 manifestModules。

然后下面根据 GlideModule 的信息去配置 Glide，这里先不分析，Glide 对象真正创建的地方是这一句：


```java
Glide glide = builder.build(applicationContext);
```

创建完之后赋值给 Glide.glide 静态对象：


```java
Glide.glide = glide;
```

而 builder 的实现类是 GlideBuilder。下面简单看看 GlideBuilder#build 方法：

```java
 @NonNull
  Glide build(@NonNull Context context) {
    //...
    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);

    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptions.lock(),
        defaultTransitionOptions,
        defaultRequestListeners,
        isLoggingRequestOriginsEnabled);
  }
```

这时候再看回一开始说到的 Glide#getRetriever 方法：

```java
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
 	//...
    return Glide.get(context).getRequestManagerRetriever();
  }
```

这时候就明白了，Glide 通过 Glide#get 方法创建实例， 并且创建了 RequestManagerRetriever 实例，这个方法的 getRequestManagerRetriever 其实就是在 GlideBuilder#Build 里面创建的。


分析完 Glide 对象是如何创建的，下面再看回 with 方法中，_getRetriever_(context)#get(context) 方法：

```java
  public RequestManager get(@NonNull Context context) {
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }
    return getApplicationManager(context);
  }
```
最终 get 方法都会调用 fragmentGet 或者 supportFragmentGet 方法，看方法名知道，这里面跟 Fragment 有关，区别就是使用的是普通的 Fragment 还说 v4 包中的 Fragment。这里看 supportFragmentGet 方法：


```java
private RequestManager supportFragmentGet(
    @NonNull Context context,
    @NonNull FragmentManager fm,
    @Nullable Fragment parentHint,
    boolean isParentVisible) {
    SupportRequestManagerFragment current =
        getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
        // TODO(b/27524013): Factor out this Glide.get() call.
        Glide glide = Glide.get(context);
        requestManager =
            factory.build(
            glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
        current.setRequestManager(requestManager);
    }
    return requestManager;
}

private SupportRequestManagerFragment getSupportRequestManagerFragment(
    @NonNull final FragmentManager fm, @Nullable Fragment parentHint, boolean isParentVisible) {
    SupportRequestManagerFragment current =
        (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
        current = pendingSupportRequestManagerFragments.get(fm);
        if (current == null) {
            current = new SupportRequestManagerFragment();
            current.setParentFragmentHint(parentHint);
            if (isParentVisible) {
                current.getGlideLifecycle().onStart();
            }
            pendingSupportRequestManagerFragments.put(fm, current);
            fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
            handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
        }
    }
    return current;
}
```

可以看到，在 supportFragmentGet 中，通过 getSupportRequestManagerFragment 方法创建了一个叫做 SupportRequestManagerFragment 的 Fragment，**这个 Fragment 是跟 Glide 的生命周期绑定有关的**。  
然后判断 requestManager 是否为空，当然，第一次的时候是空的，当空的时候，通过 factory#build 方法创建 RequestManager 对象然后通过 setRequestManager 方法设置给 Fragment ，最后再返回，这里的 factory 是 RequestManagerFactory，它的默认实现是 _DEFAULT_FACTORY：_

```java
public interface RequestManagerFactory {
    @NonNull
    RequestManager build(
        @NonNull Glide glide,
        @NonNull Lifecycle lifecycle,
        @NonNull RequestManagerTreeNode requestManagerTreeNode,
        @NonNull Context context);
}

private static final RequestManagerFactory DEFAULT_FACTORY = new RequestManagerFactory() {
    @NonNull
    @Override
    public RequestManager build(@NonNull Glide glide, @NonNull Lifecycle lifecycle,
                                @NonNull RequestManagerTreeNode requestManagerTreeNode, @NonNull Context context) {
        return new RequestManager(glide, lifecycle, requestManagerTreeNode, context);
    }
};
```

这里重点看一下  SupportRequestManagerFragment。

```java
public class SupportRequestManagerFragment extends Fragment {
  private static final String TAG = "SupportRMFragment";
  private final ActivityFragmentLifecycle lifecycle;
  //...
    @Override
  public void onStart() {
    super.onStart();
    lifecycle.onStart();
  }

  @Override
  public void onStop() {
    super.onStop();
    lifecycle.onStop();
  }

  @Override
  public void onDestroy() {
    super.onDestroy();
    lifecycle.onDestroy();
		//...
  }
}
```
在 Fragment 里面，有一个 ActivityFragmentLifecycle 对象，实现的是 Lifecycle 接口。然后在生命周期方法里面通过 ActivityFragmentLifecycle 实现了生命周期的监听。

```java
class ActivityFragmentLifecycle implements Lifecycle {
  private final Set<LifecycleListener> lifecycleListeners =
      Collections.newSetFromMap(new WeakHashMap<LifecycleListener, Boolean>());
  private boolean isStarted;
  private boolean isDestroyed;

  @Override
  public void addListener(@NonNull LifecycleListener listener) {
    lifecycleListeners.add(listener);

    if (isDestroyed) {
      listener.onDestroy();
    } else if (isStarted) {
      listener.onStart();
    } else {
      listener.onStop();
    }
  }

  @Override
  public void removeListener(@NonNull LifecycleListener listener) {
    lifecycleListeners.remove(listener);
  }

  void onStart() {
    isStarted = true;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStart();
    }
  }

  void onStop() {
    isStarted = false;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStop();
    }
  }

  void onDestroy() {
    isDestroyed = true;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onDestroy();
    }
  }
}
```

**总结来说，Glide 是通过创建一个 Fragment 绑定到当前 Activity，然后通过 Fragment 的生命周期回调去实现生命周期绑定的。**
