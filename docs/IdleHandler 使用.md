消息机制中 ，IdleHandler 可能会被忽略，但是它却有很大作用。同时在面试中也被问到了，这里做一下记录。


**用法：**

```java
//getMainLooper().myQueue()或者Looper.myQueue()
Looper.myQueue().addIdleHandler(new IdleHandler() {  
    @Override  
    public boolean queueIdle() {  
        //你要处理的事情
        return false;    
    }  
});
```

# 1. IdleHandler 作用
> **IdleHandler 可以用来提升性能，每次消费掉一个有效message，在获取下一个message时，如果当前时刻没有需要消费的有效(需要立刻执行)的message，那么会执行IdleHandler一次，执行完成之后线程进入休眠状态，直到被唤醒，不过最好不要做耗时操作。**


**换句话说，就是当在looper里面的message暂时处理完了，这个时候会回调这个接口。**

```java
    /**
     * 当前队列将进入阻塞等待消息时调用该接口回调，即队列空闲
     */
    public static interface IdleHandler {
        /**
         * 返回true就是单次回调后不删除，下次进入空闲时继续回调该方法，false只回调单次
         */
        boolean queueIdle();
    }
```

# 2. 源码解析
IdleHandler 是 MessageQueue 里面的接口，通过 getMainLooper().myQueue()或者Looper.myQueue() 的 addIdleHandler 方法，将 IdleHandler 添加到一个 ArrayList 中：


```java
public final class MessageQueue {
    //...
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
	
    //...
        
    public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }
}
```

MessageQueue 中通过 next() 方法查找消息，如果找到就返回，在 next() 方法的后半段可以看到下面代码：

```java
Message next() {
    //...
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    for (;;) {
        synchronized (this) {
            //...
            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

上面代码省略了查找消息的部分，首先看到的是通过  pendingIdleHandlerCount 判断 mIdleHandlers 中是否存在着 IdleHandler ,如果不存在，就阻塞着然后返回，如果存在，则遍历 mIdleHandlers，并执行 queueIdle 方法。queueIdle 的返回值 keep，如果是 false 的话，执行完就从 mIdleHandlers 中移除，否则保留，最后初始化 pendingIdleHandlerCount 。

# 3. 使用场景
下面举几个使用场景例子。

**想要在某个Activity绘制完成去做一些事情，那这个时机是什么时候呢？**

看到这需求，第一时间可能想到的是在 onStart 或者 onResume 回调里面去做。虽然在这两个方法中，Activity 已经在前台并且可以交互了，但是却并没有绘制完成。一个很明显的例子是你在这两个回调里面获取 View 的宽高时，是获取不到的。那么这时候就可以使用 IdleHandler 了。


其他使用场景：
> 这种特殊的handler可以实现一些很酷的事情，比如，ActivityThread中的GC和非前台Activity的生命周期管理，分别由ActivityThread的GcIdler和Idler实现。
> **1.GC:**
AMS会在某些情况下将会调用 scheduleAppGcsLocked，这个方法将会可能会引发 ActivityThread 进行两种场景的GC。一种是低内存，一种是进程运行在后台。> 低内存时会回调进程中的 Activity 的 onLowMemory，然后gc。
> 

> 运行在后台时相对没有那么严重，这个时候就使用到 IdleHandler，MessageQueue 闲置后再gc
> 

> 注意这里的gc指的都是手动调用 runtime.gc()。
> 

> 基于 ActivityThread 中的 GCWatcher 机制，ActivityThread 能感知到gc

> 不管哪种gc，ActivityThread都能在 gc 的同时选择释放一些 Activity
> 

> **2.非前台Activity的生命周期管理：**

> 这里指的是

> 1. 对 pause 的 Activity 进行 stop 或 destroy

> 2. 对 Finishing 的 Activity 进行 destroy
> 

> 例如前者，我们知道启动 Activity 时目标 Activity 的 resume 总是先于前一个 Activity 的 stop
ActivitThread 当 resume 完指定的 Activity 后也会进入 idle 状态，
> 因此这时就可以通过 IdleHanler 去调用 AMS 的 activityIdle 方法，

> activityIdle 会对整个系统进行检查，大多数 stop 和 destroy 操作都由这里引发。


> IdleHandler 在业务上也有作用，比如glide中用来在idle的时机清除已经被回收的弱引用，另外我们也可以使用它来在界面绘制完成(注意这个时机晚于onResume)之后做一些操作，比如这篇[文章](https://blog.csdn.net/tencent_bugly/article/details/78395717)中将复杂的界面设置放在绘制完成之后，这样可以先把简单的部分先显示给用户，而不会因为一部分的界面原因导致整个界面长时间处于空白状态

