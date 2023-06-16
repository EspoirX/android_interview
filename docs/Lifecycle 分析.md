# 基本使用

```java
viewModel
    .getLiveData()
    .observe(this, new Observer<String>() {
                 @Override
                 public void onChanged(@Nullable String string) {

                 }
             });
```

在 Activity 或者 Fragment 中，进场这样使用 LiveData，Observer 方法中第一个参数是 LifecycleOwner，它是一个接口，传入 this，证明 Activity 或者 Fragment 实现了它。

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    //...
}
```

找到我们继承的 SupportActivity：

```java
public class SupportActivity extends Activity implements LifecycleOwner, Component {
	//...
}
```

# 生命周期感知
Lifecycle 是如何实现生命周期感知的，可以看 SupportActivity 的 onCreate 方法：

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
}

public static void injectIfNeededIn(Activity activity) {
    android.app.FragmentManager manager = activity.getFragmentManager();
    if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
        manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
        manager.executePendingTransactions();
    }
}
```

在 onCreate 中，通过给当前 Activity 添加一个没有布局的 ReportFragment 去做生命周期监听操作。

# ReportFragment
在 ReportFragment 中，每个生命周期的方法回调都分别对应调用下面的两个方法

```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    dispatchCreate(mProcessListener);
    dispatch(Lifecycle.Event.ON_CREATE);
}
//...
public enum Event {
    ON_CREATE,
    ON_START,
    ON_RESUME,
    ON_PAUSE,
    ON_STOP,
    ON_DESTROY,
    ON_ANY
}
```

Event 是一个枚举，里面定分别对应这各个回调方法的类型。mProcessListener 的类型是 ActivityInitializationListener 监听器。dispatchXXX 的实现就是回调监听器对应的方法：

```java
private void dispatchCreate(ActivityInitializationListener listener) {
    if (listener != null) {
        listener.onCreate();
    }
}

private void dispatchStart(ActivityInitializationListener listener) {
    if (listener != null) {
        listener.onStart();
    }
}

private void dispatchResume(ActivityInitializationListener listener) {
    if (listener != null) {
        listener.onResume();
    }
}
//...
```

## dispatch
先看 dispatch 方法做了什么：

```java
private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }

    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```
分别判断 Activity 实现了 LifecycleRegistryOwner 和 LifecycleOwner 分别的操作，因为 LifecycleRegistryOwner 已经废弃掉了，所以看 LifecycleOwner 即可。

上面分析知道，SupportActivity 实现了 LifecycleOwner，所以看 LifecycleOwner 的 getLifecycle 方法实现：

```java
private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

public Lifecycle getLifecycle() {
    return this.mLifecycleRegistry;
}
```

所以 dispatch 最后会调用 LifecycleRegistry 的 handleLifecycleEvent 方法，传入的是当前生命周期回调对应的 Event 枚举对应的类型。

```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}

private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}
```

在 Lifecycle 中，除了 Event 枚举，还定义了一个 State 枚举：

```java
public enum State {
    DESTROYED,
    INITIALIZED,
    CREATED,
    STARTED,
    RESUMED;

    public boolean isAtLeast(@NonNull State state) {
        return compareTo(state) >= 0;
    }
}

static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}
```

可以看到 getStateAfter 方法就是把 Event 的类型对应的转换成 State 类型，而 getStateAfter 获取的是 **即将的事件。比如当前执行了 ONCREATE 和 ONSTOP，那么状态就会处于 CREATED。 **
**
**![640.png](https://cdn.nlark.com/yuque/0/2019/png/450005/1571278900759-d9140ff6-c102-491a-b209-0ad2cda8e5c5.png#align=left&display=inline&height=420&originHeight=420&originWidth=948&size=22319&status=done&width=948)**
**
**
mState 是一个类型变量，用来存储当前类型，在 moveToState 方法中，首先判断**如果当前所处的状态和即将要处于的状态一样就不做任何操作**，否则执行下一步。

先看看一些变量代表什么意思：
**mHandlingEvent** ：是否正在处理 Event 事件
**mAddingObserverCounter**：正在添加 Observer 计数器
**mNewEventOccurred**：是否有新的事件发生了

知道这变量意思后，应该就知道 moveToState 中那几个判断的意思了，则如果有新的事件在处理，则返回，什么都不做，如果没有，则处理新的事件。

# sync()
所以接下来看处理事件的方法 sync()

```java
private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
              + "new events from it.");
        return;
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
            && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}

```

mObserverMap 是一个缓存 map，eldest 和 newest 分别代表获取 添加的最旧和最新的条目，可能返回 null。

sync 方法根据当前状态和 mObserverMap 中的 eldest 和 newest 的状态做对比 ，判断当前状态是向前还是向后，比如由 **STARTED 到 RESUMED 是状态向前**，反过来就是状态向后，这个不要和 Activity 的生命周期搞混。

以向前为例：

```java
private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
        mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
            popParentState();
        }
    }
}
```

** ObserverWithState observer = entry.getValue(); **从 mObserverMap 中获取 ObserverWithState。

popParentState 和 pushParentState 分别是向 mParentStates 中添加和删除 item。

主要看这句：
```java
observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
```

upEvent 方法获取的是当前状态的向前状态：

```java
private static Event upEvent(State state) {
    switch (state) {
        case INITIALIZED:
        case DESTROYED:
            return ON_CREATE;
        case CREATED:
            return ON_START;
        case STARTED:
            return ON_RESUME;
        case RESUMED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}
```

observer 即是 ObserverWithState：

```java
static class ObserverWithState {
    State mState;
    GenericLifecycleObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.getCallback(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```

故名思议，ObserverWithState 就是一个包含了 State 和 LifecycleObserver 的类，而 GenericLifecycleObserver 是一个接口，它继承了 LifecycleObserver 并定义了 onStateChanged 方法，dispathcEvent 就是调用了这个方法。

GenericLifecycleObserver 接口有很多个实现类：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/450005/1571280060937-bd7514f5-c2ca-4660-bd7a-50f12c4ca6ee.png#align=left&display=inline&height=291&originHeight=223&originWidth=572&size=94817&status=done&width=746)

主要看下 ReflectiveGenericLifecycleObserver：

```java
class ReflectiveGenericLifecycleObserver implements GenericLifecycleObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```

onStateChanged 方法调用了 CallbackInfo 的 invokeCallbacks 方法。而 CallbackInfo 是通过 ClassesInfoCache 的 getInfo 方法创建的，在 getInfo 方法内部又调用了 createInfo 创建：

```java
CallbackInfo getInfo(Class klass) {
    CallbackInfo existing = mCallbackMap.get(klass);
    if (existing != null) {
        return existing;
    }
    existing = createInfo(klass, null);
    return existing;
}

private CallbackInfo createInfo(Class klass, @Nullable Method[] declaredMethods) {
    Class superclass = klass.getSuperclass();
    Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
  	//...
    Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
    boolean hasLifecycleMethods = false;
    for (Method method : methods) {
        OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
        if (annotation == null) {
            continue;
        }
        hasLifecycleMethods = true;
        Class<?>[] params = method.getParameterTypes();
        int callType = CALL_TYPE_NO_ARG;
        if (params.length > 0) {
            callType = CALL_TYPE_PROVIDER;
            if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                throw new IllegalArgumentException(
                    "invalid parameter type. Must be one and instanceof LifecycleOwner");
            }
        }
        Lifecycle.Event event = annotation.value();
 		//...
        MethodReference methodReference = new MethodReference(callType, method);
        verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
    }
    CallbackInfo info = new CallbackInfo(handlerToEvent);
    mCallbackMap.put(klass, info);
    mHasLifecycleMethods.put(klass, hasLifecycleMethods);
    return info;
}
```

getInfo 方法用到了缓存，因为 createInfo 里面有反射操作。首先看：

**OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);**
**
在 for 循环中，不断的遍历各个方法，获取方法上的名为 OnLifecycleEvent 的注解，这个注解正是实现 LifecycleObserver 接口时用到的。接下来：

**Lifecycle.Event event = annotation.value();**

拿到 **OnLifecycleEvent **注解的值，也就是在 @OnLifecycleEvent 中定义的事件。

接下来 verifyAndPutHandler 方法：

```java
private void verifyAndPutHandler(Map<MethodReference, Lifecycle.Event> handlers,
                                 MethodReference newHandler, Lifecycle.Event newEvent, Class klass) {
    Lifecycle.Event event = handlers.get(newHandler);
    if (event != null && newEvent != event) {
        Method method = newHandler.mMethod;
        throw new IllegalArgumentException(
            "Method " + method.getName() + " in " + klass.getName()
            + " already declared with different @OnLifecycleEvent value: previous"
            + " value " + event + ", new value " + newEvent);
    }
    if (event == null) {
        handlers.put(newHandler, newEvent);
    }
}
```

该方法用于将 MethodReference 和对应的 Event 存在类型为 Map<MethodReference, Lifecycle.Event>  的 handlerToEvent 中。

最后创建 CallbackInfo ，并将 handlerToEvent 传进去。

创建完 CallbackInfo，接下来看它的 invokeCallbacks 方法：

```java
static class CallbackInfo {
    final Map<Lifecycle.Event, List<MethodReference>> mEventToHandlers;
    final Map<MethodReference, Lifecycle.Event> mHandlerToEvent;

    CallbackInfo(Map<MethodReference, Lifecycle.Event> handlerToEvent) {
        mHandlerToEvent = handlerToEvent;
        mEventToHandlers = new HashMap<>();
        for (Map.Entry<MethodReference, Lifecycle.Event> entry : handlerToEvent.entrySet()) {
            Lifecycle.Event event = entry.getValue();
            List<MethodReference> methodReferences = mEventToHandlers.get(event);
            if (methodReferences == null) {
                methodReferences = new ArrayList<>();
                mEventToHandlers.put(event, methodReferences);
            }
            methodReferences.add(entry.getKey());
        }
    }

    @SuppressWarnings("ConstantConditions")
    void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
        invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
        invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
                              target);
    }

    private static void invokeMethodsForEvent(List<MethodReference> handlers,
                                              LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
        if (handlers != null) {
            for (int i = handlers.size() - 1; i >= 0; i--) {
                handlers.get(i).invokeCallback(source, event, mWrapped);
            }
        }
    }
}
```

在构造函数中，通过 for 循环将 handlerToEvent 进行数据类型转换，转化为一个 HashMap，key 的值为事件，value 的值为 MethodReference。

invokeMethodsForEvent 方法会传入 mEventToHandlers.get(event)，也就是事件对应的 MethodReference 的集合。invokeMethodsForEvent 方法中会遍历 MethodReference 的集合，调用 MethodReference 的 invokeCallback 方法。

```java
static class MethodReference {
    final int mCallType;
    final Method mMethod;

    MethodReference(int callType, Method method) {
        mCallType = callType;
        mMethod = method;
        mMethod.setAccessible(true);
    }

    void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
        //noinspection TryWithIdenticalCatches
        try {
            switch (mCallType) {
                case CALL_TYPE_NO_ARG:
                    mMethod.invoke(target);
                    break;
                case CALL_TYPE_PROVIDER:
                    mMethod.invoke(target, source);
                    break;
                case CALL_TYPE_PROVIDER_WITH_EVENT:
                    mMethod.invoke(target, source, event);
                    break;
            }
        } catch (InvocationTargetException e) {
            throw new RuntimeException("Failed to call observer method", e.getCause());
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }
  //...
}
```

MethodReference 类中有两个变量，一个是 callType，它代表调用方法的类型，在 createInfo 中赋值，另一个是 Method，它代表方法，不管是哪种 callType 都会通过 invoke 对方法进行反射。

简单来说，实现 LifecycleObserver 接口的类中，注解修饰的方法和事件会被保存起来，通过反射对事件的对应方法进行调用。

![6402.png](https://cdn.nlark.com/yuque/0/2019/png/450005/1571281158187-dbb517e8-0c41-4e14-9e7e-43c0c01227d5.png#align=left&display=inline&height=744&originHeight=744&originWidth=1014&size=30973&status=done&width=1014)

# LiveData

回到最开始的 LiveData 的 observe 方法：

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                                           + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```

通过上面分析，** owner.getLifecycle().addObserver(wrapper); **方法调用的是 LifecycleRegistry 的 addObserver 方法。

```java
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    if (previous != null) {
        return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }

    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}
```

在 addObserver 中，创建 ObserverWithState，然后往 mObserverMap 中添加，putIfAbsent 方法会判断如果已经在 map 中存在就直接返回，否则添加到 map 中，返回 null。所以接下来判断的是如果存在 map 中，则返回。
然后下面再走 dispatchEvent 和 sync 逻辑。

回到 LiveData 的 observe 方法，知道 LifecycleBoundObserver 肯定是实现了 LifecycleObserver 接口的：

```java
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

经过上面的分析，知道会回调 onStateChanged 方法，然后调用 activeStateChanged 方法：

```java
private abstract class ObserverWrapper {
    final Observer<T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;

    ObserverWrapper(Observer<T> observer) {
        mObserver = observer;
    }

    abstract boolean shouldBeActive();

    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }

    void detachObserver() {
    }

    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        mActive = newActive;
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        LiveData.this.mActiveCount += mActive ? 1 : -1;
        //过去是inactive，现在是active
        if (wasInactive && mActive) {
            onActive();
        }
        //过去没有订阅，并且现在是inactive
        if (LiveData.this.mActiveCount == 0 && !mActive) {
            onInactive();
        }
        if (mActive) {
            dispatchingValue(this);
        }
    }
}
```

activeStateChanged 首先判断新来的状态和旧状态是否相同，相同则忽略，然后判断 LiveData上 的活跃态的数量是否为 0，为 0 说明之前处于 Inactive，然后统计现在的订阅数，接着就是三个 if 判断，注释在代码里。正式这三个判断，LiveData 可以接收到 onActive 和 onInactive 的回调。
dispatchingValue(this) 是当状态变为 active 时调用，用来更新数据。里面会用到 considerNotify 方法。

```java
private void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                 mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
```
 
**considerNotify**
**
```java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
```

可以看到最后回调了 onChanged 方法，对应最开头的使用例子：

```java
viewModel
    .getLiveData()
    .observe(this, new Observer<String>() {
                 @Override
                 public void onChanged(@Nullable String string) {

                 }
             });
```


