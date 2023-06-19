lifecycle 是基于观察者模式的，整体的话分为几部分：

1.  生命周期的监听
2. Event 和 State 的关系
3. 注册和发送流程
4. 响应流程


# 生命周期的监听

要实现 lifecycle ，首先要在我们的 Activity 中实现 LifecycleOwner 接口，在 getLifecycle 方法中返回 
LifecycleRegistry 实例，通过 LifecycleRegistry 的 handleLifecycleEvent 方法在各个生命周期中监听生命周期。

然后用一个类实现 LifecycleObserver 接口用来响应监听。通过 LifecycleRegistry 的 addObserver 方法添加到监听队列中。
在 Activity 中，系统通过往 activity 里面添加一个 Fragment 然后通过监听 Fragment 的生命周期来实现了这一过程。

# Event 和 State 的关系
State 是当前 Lifecycle 的状态值，而 Event 是 Lifecycle 接下去的动作值。

![lifecycle3.jpeg](https://s2.loli.net/2023/06/19/5LjWmilQoaZpwqC.png)

State 是 中间那行，Event 是上下 ON_ 对应的动作。

![lifecycle4.jpeg](https://s2.loli.net/2023/06/19/EANo6X3bncUVDSi.png)

看到上面这个图片，我们的 onPause 被我们改名字了，改成了 onStart2，当然实际上它跟上面的 onStart 是有区别的，实质上它是 onPause，那我们把名字改了，还能知道吗？？？答案是可以的，为什么？因为有 Event 配合，所以就能知道。
比如：

比如原来房价为13000一平米，涨了1000，变成了14000，再涨了1000变成了15000，这时候跌了1000，又变回了14000，请问这时候二个14000我可以做区分吗？？？当然可以，配合动作值就可以，因为一个叫做上涨1000的14000， 一个叫跌了1000的14000。

![lifecycle5.webp](https://s2.loli.net/2023/06/19/LZ8tVIQGj1w24Pm.webp)

当然为了好看点，上面的房价变化我们可以写成：

![lifecycle6.webp](https://s2.loli.net/2023/06/19/kHGKDirgpsq1bOx.webp)

同理我们的 onPause 虽然名字改成了 onStart, 但是因为是通过 ON_PAUSE 动作值配合，我们知道这时候的onStart 其实是 onPuase，   onStop 更改为 onCreate 也是一样的道理。

# 注册和发送流程

## 注册过程
整个 lifecycle 核心类是 LifecycleRegistry，它继承类 Lifecycle。在 LifecycleRegistry 中，有一个 mObserverMap 的 map 队列，添加的 LifecycleObserver 会加入进来。

注册的主要方法是 addObserver

```java
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

首先获取一下当前的状态 initialState，刚开始 LifecycleObserver 的默认状态是 INITIALIZED，  
但如果 LifecycleRegistry 的当前状态是 DESTROYED，就业直接把新添加的 LifecycleObserver 也变成 DESTROYED,后续很多逻辑也就走不通了，就好比 Activity 已经变成了 onDestory 了，也不可能在变成其他什么 onCreate，onResume 状态了.

ObserverWithState 是包装了 LifecycleObserver 和 State 的类。

然后添加到 mObserverMap 中，注意用的是 putIfAbsent 方法，如果已经存在则会直接返回，不会覆盖，所以如果 previous 不为 null，直接返回，则证明已经添加过，不需要再添加了。

然后获取目标状态值  targetState

通过 while 把当前的加入的观察者的 State 值，与目标值进行比较（我们当前的值，一开始被赋予了INITIALIZED 或者 DESTROYED）  
在 while 循环里面 通过 pushParentState 方法先把它自身的状态存起来，然后通过 dispatchEvent 方法分发 Event。**参数是根据当前的观察者的状态值的上一步Event（实现一步步提升，一步步分发），**改变观察者的状态后，通过 popParentState 方法原来存的状态给删除掉

最后在重新计算目标状态值 targetState

如果没有重入，调用 sync 方法同步整个队列。


## 发送事件过程
从上面我们知道，在生命周期回调中，通过 LifecycleRegist 的 handleLifecycleEvent 方法处理事件

```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    //获取 下一个 State 值，根据 State 和 Event 的关系图
    State next = getStateAfter(event);
    moveToState(next); //然后直接移动到指定的 State 状态值
}
```

moveToState：

```java
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

moveToState 会进行一些判断，如果当前的 mState 和要求变化的 state 相同，则不往下执行  
如果当前正在进行 sync 同步，或者同时添加了多个观察者，不往下执行，否则进行 sync 更新队列。

### sync()

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

通过 while 循环，判断条件是是否完成同步，根据当前状态和 mObserverMap 中的 eldest 和 newest 的状态做对比 ，判断当前状态是向前还是向后，比如由 **STARTED** 到 **RESUMED** 是**状态向前**，反过来就是状态向后。

**后退操作**是从**队列头开始，往队列尾更新，向前操作相反**
 
backwardPass 和 forwardPass 方法中调用 ObserverWithState 的 dispatchEvent 方法进行事件分发。

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

这里调用了我们封装的 ObserverWithState 对象里面的 **mLifecycleObserver 的 onStateChanged 方 法**，而这个 mLifecycleObserver 是把我们 addObserver 时候传入的我们自己的 Observer 通过 Lifecycling.  getCallback 方法再次处理后返回了一个新的 Observer。说明我们**传入的不同的 Observer，返回的这个回调的mLifecycleObserver 不同。**

事件通过这里分发出去了。


# LiveData
livedata 可以看成是封装了上面许多步骤的类，LiveData 里面实现 LifecycleObserver 的类是 LifecycleBoundObserver，所以得知最终事件会分发到 onStateChanged 方法里面。最后会一层层调用回调到 onChanged 方法里面。

而 livedata 的 observe 方法其实就是往 LifecycleRegistry 里面注册类一个事件，是通过 addObserver 实现的。
