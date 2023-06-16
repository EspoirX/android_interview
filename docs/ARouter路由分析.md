# 路由与组件化的关系
路由天生契合组件化。并不是说路由是为组件化而设计的，实际上两者没有任何关系。在组件化中，各个组件在开发时不会相互依赖，而是共同依赖于 base module，所以各个模块并不能直接通讯，因为并不能得到直接的引用。而路由正好解决了这个问题，所以说天生契合组件化，但并不是为组件化而设计的。

# 路由基本原理
在 ARouter 里面，会看到在一些 Activity 里面声明  注解，这个称之为路由地址：

```java
@Route(path = "/main/main")
public class MainActivity extends AppCompatActivity {
	//...
}

@Route(path = "/module1/module1main")
public class Module1MainActivity extends AppCompatActivity {
	//...
}
```

路由框架会在项目的编译器扫描所有添加 @Route 注解的 Activity 类，然后将 oute 注解中的 path 地址和Activity.class 文件一一对应保存，如直接保存在 map 中。

```java
//项目编译后通过apt生成如下方法
public HashMap<String, ClassBean> routeInfo() {
    HashMap<String, ClassBean> route = new HashMap<String, ClassBean>();
    route.put("/main/main", MainActivity.class);
    route.put("/module1/module1main", Module1MainActivity.class);
    route.put("/login/login", LoginActivity.class);
}
```

这样我们想在 app 模块的 MainActivity 跳转到 login 模块的 LoginActivity，那么便只需调用如下：

```java
//不同模块之间启动Activity
public void login(String name, String password) {
    HashMap<String, ClassBean> route = routeInfo();
    LoginActivity.class classBean = route.get("/login/login");
    Intent intent = new Intent(this, classBean);
    intent.putExtra("name", name);
    intent.putExtra("password", password);
    startActivity(intent);
}
```

# Route注解如何实现路由跳转

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.CLASS)
public @interface Route {

    //路径URL字符串
    String path();

  	//组名，默认为一级路径名；一旦被设置，跳转时必须赋值
    String group() default "";

    //该路径的名称，用于产生JavaDoc
    String name() default "";

    //额外配置的开关信息；譬如某些页面是否需要网络校验、登录校验等
    int extras() default Integer.MIN_VALUE;

    //该路径的优先级
    int priority() default -1;
}
```

这里看到 Route 注解里有 path 和 group，是对路由进行分组。因为当项目变得越来越大庞大的时候，为了便于管理和减小首次加载路由表过于耗时的问题，要所有的路由进行分组。在 ARouter 中会要求路由地址至少需要两级，如"/xx/xx"，一个模块下可以有多个分组。如下的路由注解：

```java
@Route(path = "/test/activity1")
public class MainActivity extends AppCompatActivity {}

@Route(path = "/test2/activity2")
public class Main2Activity extends AppCompatActivity {}
```

在项目编译的时候，我们将会通过apt生成 ARouter$$Group$$test 和 ARouter$$Group$$test2 文件，里面记录着分组的路由地址和 ActivityClass 映射信息。

```java
public class ARouter$$Group$$test implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/test/activity1", RouteMeta.build(RouteType.ACTIVITY, MainActivity.class, "/test1/activity", "test2", new java.util.HashMap<String, Integer>(){{put("key1", 8); }}, -1, -2147483648));
  }
}

public class ARouter$$Group$$test2 implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/test2/activity2", RouteMeta.build(RouteType.ACTIVITY, Main2Activity.class, "/test2/activity2", "test2", new java.util.HashMap<String, Integer>(){{put("key1", 8); }}, -1, -2147483648));
  }
}
```

可以看到这两个类实现了 IRouteGroup 接口并实现 loadInto 方法，里面就是给 map 添加值。

如果我们在 login_module 中想启动 app_module 中的 MainActivity 类，首先，我们已知 MainActivity 类的路由地址是 "/main/main"，第一个 "/main" 代表分组名，那么我们岂不是可以像下面这样调用去得到 MainActivity 类文件，然后 startActivity。这里的 RouteMeta 只是存有 Activity class 文件的封装类，先不用理会。

```java
public void test() {
    ARouter$$Group$$main rootApp = new  ARouter$$Group$$main();
    HashMap<String, Class<? extends IRouteGroup>> rootMap = new HashMap<>();
    rootApp.loadInto(rootMap);

    //得到/main分组
    Class<? extends IRouteGroup> aClass = rootMap.get("main");
    try {
        HashMap<String, RouteMeta> groupMap = new HashMap<>();
        aClass.newInstance().loadInto(groupMap);
        //得到MainActivity
        RouteMeta main = groupMap.get("/main/main");
        Class<?> mainActivityClass = main.getDestination();

        Intent intent = new Intent(this, mainActivityClass);
        startActivity(intent);
    } catch (InstantiationException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }
}
```

这就是路由跳转的原理。

# 源码分析
首先看 ARouter 的初始化：

```java
ARouter.init(getApplication());

public static void init(Application application) {
    if (!hasInit) {
        logger = _ARouter.logger; //持有 日志打印的 全局静态标量
        _ARouter.logger.info(Consts.TAG, "ARouter init start."); //打印 ARouter初始化日志
        hasInit = _ARouter.init(application); //移交 _ARouter去 初始化

        if (hasInit) {
            _ARouter.afterInit(); //获取 interceptorService
        }

        _ARouter.logger.info(Consts.TAG, "ARouter init over."); //打印 ARouter初始化日志
    }
}
```

可以看到在 init 方法中，初始化了 log 信息，然后交给了 _ARouter 去初始化，在 _ARouter 的 init 方法中，又调用了 LogisticsCenter.init(mContext, executor); 去初始化。

```java
public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
    mContext = context; //静态持有Application的上下文 
    executor = tpe;  //静态持有 线城池

    try {
        long startInit = System.currentTimeMillis();
        //billy.qi modified at 2017-12-06
        //load by plugin first
        loadRouterMap();
        if (registerByPlugin) {
            logger.info(TAG, "Load router map by arouter-auto-register plugin.");
        } else {
            Set<String> routerMap;

            // It will rebuild router map every times when debuggable.
            if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
                // These class was generated by arouter-compiler.
                routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                if (!routerMap.isEmpty()) {
                    context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                }

                PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
            } else {
                logger.info(TAG, "Load router map from cache.");
                routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
            }

            logger.info(TAG, "Find router map finished, map size = " + routerMap.size() + ", cost " + (System.currentTimeMillis() - startInit) + " ms.");
            startInit = System.currentTimeMillis();

            for (String className : routerMap) {
                if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                    // This one of root elements, load root.
                    ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                    // Load interceptorMeta
                    ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                    // Load providerIndex
                    ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                }
            }
        }

        logger.info(TAG, "Load root element finished, cost " + (System.currentTimeMillis() - startInit) + " ms.");

        if (Warehouse.groupsIndex.size() == 0) {
            logger.error(TAG, "No mapping files were found, check your configuration please!");
        }

        if (ARouter.debuggable()) {
            logger.debug(TAG, String.format(Locale.getDefault(), "LogisticsCenter has already been loaded, GroupIndex[%d], InterceptorIndex[%d], ProviderIndex[%d]", Warehouse.groupsIndex.size(), Warehouse.interceptorsIndex.size(), Warehouse.providersIndex.size()));
        }
    } catch (Exception e) {
        throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
    }
}
```

在 init 方法中，首先通过指定包名 com.alibaba.android.arouter.routes（ROUTE_ROOT_PAKCAGE）找到所有编译期生成的 routes 目录下的类名，保存在 routerMap 下，然后再保存在 sp 文件中。然后再通过判断分别调用生成的类的 loadInto 接口方法，注意传入的参数是在 Warehouse 下的 map，这些 map 是静态保存的。Warehouse 称为内存仓库。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/450005/1569741871919-3e44f286-b6b5-4c36-b40f-f7f926ee204d.png#align=left&display=inline&height=142&originHeight=284&originWidth=810&size=140832&status=done&width=405)

简单来说，初始化就是找到如图上面那些类，并调用里面的 loadInto 方法存储信息到内容仓库中，其中 内存仓库Warehouse缓存了全局应用的【组别的清单列表】、【Ioc的动作路由清单列表】、【模块内的拦截器清单列表】，3个以 _index 为结尾的 Map 对象。

```java
class Warehouse {
    // Cache route and metas
    static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>();//【组别的清单列表】 包含了组名与对应组内的路由清单列表Class的映射关系(这里只存储了未导入到 routes在键盘每个的组)
    static Map<String, RouteMeta> routes = new HashMap<>();//【组内的路由清单列表】包含了对应分组下的，路由URL与目标对象Class的映射关系；

    // Cache provider
    static Map<Class, IProvider> providers = new HashMap<>(); //缓存 IOC  目标class与已经创建了的对象 TODO ?全局应用共享一个IOc依赖注入对象？
    static Map<String, RouteMeta> providersIndex = new HashMap<>();//【Ioc的动作路由清单列表】包含了使用依赖注入方式的某class的  路由URL 与class映射关系

    // Cache interceptor
    //【模块内的拦截器清单列表】包含了某个模块下的拦截器 与 优先级的映射关系
    static Map<Integer, Class<? extends IInterceptor>> interceptorsIndex = new UniqueKeyTreeMap<>("More than one interceptors use same priority [%s]");
    static List<IInterceptor> interceptors = new ArrayList<>();//已排序的拦截器实例对象
    //```
}
```

# ARouter 运行时 API 调用过程分析
```java
ARouter.getInstance()
    .build("/test/activity2")
    .navigation();
```

ARouter 是单例模式， build 方法的实现交给了代理类 _ARouter，最终返回的是一个 Postcard 对象：


```java
public Postcard build(String path) {
    return _ARouter.getInstance().build(path);
}
protected Postcard build(String path) {
    if (TextUtils.isEmpty(path)) {
        throw new HandlerException(Consts.TAG + "Parameter is invalid!");
    } else {
        //通过 ARouter 的 Ioc 方式(IProvider的ByType())方式找到  动态修改路由类
        PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
        if (null != pService) {
            //如果全局应用有实现 PathReplaceService.class 接口，
            //则执行 “运行期动态修改路由”逻辑。生成转换后的路由。
            path = pService.forString(path);
        }
        return build(path, extractGroup(path));
    }
}
```

其实整个build的过程分为两个顺序部分：

1. 使用 Ioc byType() 方式寻找 PathReplaceService.class 接口的实现类，实现 “运行期动态修改路”
2. 正常的本次路由导航

## 运行期动态修改路由PathReplaceService的实现(IOC 也就是IProvider.byType() = navigation(class)的实现方式，用于获取 路由目标实例)

在 build 方法中，可以看到调用了 ARouter 的 navigation 方法来获取 pService：

```java
public <T> T navigation(Class<? extends T> service) {
    return _ARouter.getInstance().navigation(service);
}

protected <T> T navigation(Class<? extends T> service) {
    try {
        Postcard postcard = LogisticsCenter.buildProvider(service.getName());

        // Compatible 1.0.5 compiler sdk.
        // Earlier versions did not use the fully qualified name to get the service
        if (null == postcard) {
            // No service, or this service in old version.
            postcard = LogisticsCenter.buildProvider(service.getSimpleName());
        }

        if (null == postcard) {
            return null;
        }

        LogisticsCenter.completion(postcard);
        return (T) postcard.getProvider();
    } catch (NoRouteFoundException ex) {
        logger.warning(Consts.TAG, ex.getMessage());
        return null;
    }
}

public static Postcard buildProvider(String serviceName) {
    RouteMeta meta = Warehouse.providersIndex.get(serviceName);
    if (null == meta) {
        return null;
    } else {
        return new Postcard(meta.getPath(), meta.getGroup());
    }
}
```

可以看到，最终实现在 buildProvider 中，从内存仓库的【Ioc的动作路由清单列表】中找到，对应 Name 对应的 路由元信息，然后 根据路由元信息  生成 Postcard对象，赋值其 路径URL 和 组名 信息。如果找不到就返回 null。如果找到就会调用  LogisticsCenter.completion(postcard); 来完善 postcard 对象。

```java
public synchronized static void completion(Postcard postcard) {
    if (null == postcard) {
        throw new NoRouteFoundException(TAG + "No postcard!");
    }
	//根据路径URL获取到路径元信息
    RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
    //如果未获取到路径元信息，可能是由于 未加载对应分组的【组内清单列表】 or 的确没有
    if (null == routeMeta) {    // Maybe its does't exist, or didn't load.
        //从【组别的清单列表】拿到对应组的 组内清单创建逻辑
        Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
        //如果为空，则丢出异常，未找到
        if (null == groupMeta) {
            throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
        } else {
            // Load route and cache it into memory, then delete from metas.
            try {
                if (ARouter.debuggable()) {
                    logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] starts loading, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                }
				//实例化【组内清单创建逻辑】
                IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                //将该组的【组内清单列表】加入到内存仓库中
                iGroupInstance.loadInto(Warehouse.routes);
                //从【组别的清单列表】移除当前组
                Warehouse.groupsIndex.remove(postcard.getGroup());

                if (ARouter.debuggable()) {
                    logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] has already been loaded, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                }
            } catch (Exception e) {
                throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
            }

            completion(postcard);   // Reload 再次触发完善逻辑，这次会走 else 逻辑
        }
    } else {
        postcard.setDestination(routeMeta.getDestination()); //目标 class
        postcard.setType(routeMeta.getType()); //路由类型
        postcard.setPriority(routeMeta.getPriority()); //路由优先级
        postcard.setExtra(routeMeta.getExtra()); //额外的配置开关信息

        Uri rawUri = postcard.getUri();
        //如果有URI，则根据路由元信息的“目标Class的需要注入的参数
        //的参数名称：参数类型TypeKind  paramsType”和“URI的?参数”进行赋值
        if (null != rawUri) {   // Try to set params into bundle.
            //“URI的?参数”
            Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
            //“目标Class的需要注入的参数 的参数名称：参数类型TypeKind ”
            Map<String, Integer> paramsType = routeMeta.getParamsType();
	
            if (MapUtils.isNotEmpty(paramsType)) {
                // Set value by its type, just for params which annotation by @Param
                //目标Class的需要注入的参数 的参数名称：参数类型TypeKind”
                //，向postcard的bundle中put对应的携带数据
                for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                    setValue(postcard,
                             params.getValue(), //参数类型TypeKind
                             params.getKey(),   //参数名称
                             resultMap.get(params.getKey())); //参数值
                }

                // Save params name which need auto inject.
                postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
            }

            // Save raw uri
            postcard.withString(ARouter.RAW_URI, rawUri.toString());
        }

        switch (routeMeta.getType()) {
            //PROVIDER类型的路由则实现 实例化目标类 + 绿色通道(byType方式的核心实现)
            case PROVIDER:  // if the route is provider, should find its instance
                // Its provider, so it must implement IProvider
                Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                IProvider instance = Warehouse.providers.get(providerMeta);
                //目标class的实例对象是否已经存在了，如果存在了就直接使用，
                //不存在才创建新的？ TODO Ioc导致全局共享一个对象？
                if (null == instance) { // There's no instance of this provider
                    IProvider provider;
                    try {
                        provider = providerMeta.getConstructor().newInstance();
                        provider.init(mContext);
                        Warehouse.providers.put(providerMeta, provider);
                        instance = provider;
                    } catch (Exception e) {
                        throw new HandlerException("Init provider failed! " + e.getMessage());
                    }
                }
                postcard.setProvider(instance); //实例化并持有目标类
                postcard.greenChannel();    //绿色通道 Provider should skip all of interceptors
                break;
            case FRAGMENT:
                postcard.greenChannel();    // Fragment needn't interceptors
            default:
                break;
        }
    }
}
```

代码比较长，注意分为两部分，就是 if 和 else 两部分。

第一部分，if 部分：
首先根据 postcard 的信息在内存仓库中查找路径信息 RouteMeta，如果找不到，则在内存仓库中查找分组信息 groupMeta。可以看到，分组信息不能为 null。找到后调用分组信息的 loadInto 方法，并存在内存仓库中，这样就有 RouteMeta 信息了，然后再移除分组信息。准备好后再次调用自己，这样就会走 else 逻辑了。

第二部分 else：
拿到 RouteMeta 后，根据 RouteMeta 信息去完善 Postcard。

# navigation()
在 build 过后拿到了 Postcard 对象，然后调用它的 navigation 方法：

```java
public Object navigation() {
    return navigation(null);
}

public Object navigation(Context context) {
    return navigation(context, null);
}

public Object navigation(Context context, NavigationCallback callback) {
    return ARouter.getInstance().navigation(context, this, -1, callback);
}

public Object navigation(Context mContext, Postcard postcard, int requestCode, NavigationCallback callback) {
    return _ARouter.getInstance().navigation(mContext, postcard, requestCode, callback);
}
```

可以看到，最终还是调用了 _ARouter 的 navigation 方法。

```java
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
    if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
        // Pretreatment failed, navigation canceled.
        return null;
    }

    try {
        //完善postcard。（之前不是已经完善过一次了吗？）
        LogisticsCenter.completion(postcard);
    } catch (NoRouteFoundException ex) {
        logger.warning(Consts.TAG, ex.getMessage());
		//弹出错误提示
        if (debuggable()) {
            // Show friendly tips for user.
            runInMainThread(new Runnable() {
                @Override
                public void run() {
                    Toast.makeText(mContext, "There's no route matched!\n" +
                                   " Path = [" + postcard.getPath() + "]\n" +
                                   " Group = [" + postcard.getGroup() + "]", Toast.LENGTH_LONG).show();
                }
            });
        }
		//执行到这里，触发查找失败
        if (null != callback) {
            callback.onLost(postcard);
        } else {
            // No callback for this invoke, then we use the global degrade service.
            DegradeService degradeService = ARouter.getInstance().navigation(DegradeService.class);
            if (null != degradeService) {
                degradeService.onLost(context, postcard);
            }
        }

        return null;
    }

    //执行到这里，说明找到了路由元信息，触发  路由查找的回调
    if (null != callback) {
        callback.onFound(postcard);
    }
	//绿色通道校验，在完善 PostCard 里面有设置过
    if (!postcard.isGreenChannel()) {   // It must be run in async thread, maybe interceptor cost too mush time made ANR.
        //调用拦截器截面控制器，遍历内存仓库的自定义拦截器，并在异步线程中执行拦截函数
        interceptorService.doInterceptions(postcard, new InterceptorCallback() {
    	  
            //根据 路由类型执行具体路由操作
            @Override
            public void onContinue(Postcard postcard) {
                _navigation(context, postcard, requestCode, callback);
            }

            /**
                 * Interrupt process, pipeline will be destory when this method called.
                 *
                 * @param exception Reson of interrupt.
                 */
            @Override
            public void onInterrupt(Throwable exception) {
                if (null != callback) {
                    callback.onInterrupt(postcard);
                }

                logger.info(Consts.TAG, "Navigation failed, termination by interceptor : " + exception.getMessage());
            }
        });
    } else {
        //执行具体操作
        return _navigation(context, postcard, requestCode, callback);
    }

    return null;
}
```

```java
private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    final Context currentContext = null == context ? mContext : context;

    switch (postcard.getType()) {
        case ACTIVITY:
            // Build intent
            final Intent intent = new Intent(currentContext, postcard.getDestination());
            intent.putExtras(postcard.getExtras());

            // Set flags.
            int flags = postcard.getFlags();
            if (-1 != flags) {
                intent.setFlags(flags);
            } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            }

            // Set Actions
            String action = postcard.getAction();
            if (!TextUtils.isEmpty(action)) {
                intent.setAction(action);
            }

            // Navigation in main looper.
            runInMainThread(new Runnable() {
                @Override
                public void run() {
                    startActivity(requestCode, currentContext, intent, postcard, callback);
                }
            });

            break;
        case PROVIDER:
            return postcard.getProvider();
        case BOARDCAST:
        case CONTENT_PROVIDER:
        case FRAGMENT:
            Class fragmentMeta = postcard.getDestination();
            try {
                Object instance = fragmentMeta.getConstructor().newInstance();
                if (instance instanceof Fragment) {
                    ((Fragment) instance).setArguments(postcard.getExtras());
                } else if (instance instanceof android.support.v4.app.Fragment) {
                    ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                }

                return instance;
            } catch (Exception ex) {
                logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
            }
        case METHOD:
        case SERVICE:
        default:
            return null;
    }

    return null;
}
```

```java
private void startActivity(int requestCode, Context currentContext, Intent intent, Postcard postcard, NavigationCallback callback) {
    if (requestCode >= 0) {  // Need start for result
        if (currentContext instanceof Activity) {
            ActivityCompat.startActivityForResult((Activity) currentContext, intent, requestCode, postcard.getOptionsBundle());
        } else {
            logger.warning(Consts.TAG, "Must use [navigation(activity, ...)] to support [startActivityForResult]");
        }
    } else {
        ActivityCompat.startActivity(currentContext, intent, postcard.getOptionsBundle());
    }

    if ((-1 != postcard.getEnterAnim() && -1 != postcard.getExitAnim()) && currentContext instanceof Activity) {    // Old version.
        ((Activity) currentContext).overridePendingTransition(postcard.getEnterAnim(), postcard.getExitAnim());
    }

    if (null != callback) { // Navigation over.
        callback.onArrival(postcard);
    }
}
```

可以看到，如果类型是 Activity，最终会调用 ActivityCompat.startActivity 方法去启动 Activity。
Fragment 的话，会通过 newInstance 去实例化，并调用 setArguments 参数传值。
如果是 Ioc，会调用 getProvider 返回 IProvider 实例。
如果是 Boardcast 或者是 ConentProvider，可以看到他们的case 中没有 break，所以他们的逻辑也是跟 Fragment 一样。
