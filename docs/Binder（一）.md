# 什么是 Binder?
 简单地说，Binder 是 Android 平台上的一种跨进程交互技术。  
 Binder机制具有两层含义：

- 是一种跨进程通信手段（IPC，Inter-Process Communication）。
- 是一种远程过程调用手段（RPC，Remote Procedure Call）。

# Binder 跨进程机制
Binder 跨进程机制可简述为：  
**Binder 代理方（BpBinder）通过 Binder 机制 到 Binder 响应方（BBinder）**  
**BpBinder 主要用于向远方发送语义，BBinder 用于响应语义**  
 
如果要起到“远程过程调用”的作用，如果要支持远程过程调用，我们还必须提供“接口代理方”和“接口实现体”

分别对应是 **BpInterface（接口代理）** 和 **BnInterface（接口实现）**  

BpBinder 被聚合进 BpInterface，但 BnInterface 是继承于 BBinder 的，它并没有采用聚合的方式来包含一个 BBinder 对象。

![binder1.png](https://s2.loli.net/2023/06/19/xuLyZg4bHtTJecr.png)

# ProcessState
在每个进程中，会有一个全局的 ProcessState 对象。这个很容易理解，ProcessState 的字面意思不就是“进程状态”吗，当然应该是每个进程一个 ProcessState。

```cpp
class ProcessState : public virtual RefBase{
public:
    static  sp<ProcessState>    self();
    . . . . . .
    void                startThreadPool();
    . . . . . .
    void                spawnPooledThread(bool isMain);
    status_t            setThreadPoolMaxThreadCount(size_t maxThreads);

private:
    friend class IPCThreadState;
    . . . . . .
    
    struct handle_entry {
        IBinder* binder;
        RefBase::weakref_type* refs;
    };
    handle_entry*       lookupHandleLocked(int32_t handle);
    int                 mDriverFD;
    void*               mVMStart;
    mutable Mutex       mLock;  // protects everything below.
    
    Vector<handle_entry> mHandleToObject;
    . . . . . .
    KeyedVector<String16, sp<IBinder> > mContexts;
    . . . . . .
};
```

Binder 内核被设计成一个驱动程序，所以 ProcessState 里专门搞了个 **mDriverFD** 域，来记录 binder 驱动对应的**句柄值**，以便随时和 binder 驱动通信。  
ProcessState 对象采用了典型的**单例模式**，在一个应用进程中，只会有 **唯一的一个** ProcessState 对象，它将被进程中的多个线程共用，因此每个进程里的线程其实是共用所打开的那个驱动句柄（mDriverFD）的，示意图如下：

![binder2.png](https://s2.loli.net/2023/06/19/X51r6aBtEOqhfQW.png)

每个进程都是这样的结构，组合起来就是 Binder 驱动

![binder3.png](https://s2.loli.net/2023/06/19/HfSnXDUiNwoRyqG.png)

ProcessState 对象构造之时，就会打开 binder 驱动：

```cpp
ProcessState::ProcessState()
    : mDriverFD(open_driver())     // 打开binder驱动。
    , mVMStart(MAP_FAILED)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    . . . . . .
    mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
    . . . . . .    
}
```


**open_driver**()，打开 /dev/binder 驱动。

```cpp
static int open_driver(){
    int fd = open("/dev/binder", O_RDWR);
    . . . . . .
    status_t result = ioctl(fd, BINDER_VERSION, &vers);
    . . . . . .
    size_t maxThreads = 15;
    result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
    . . . . . .
    return fd;
}
```


# ServiceManager

在 Android 中，系统提供的服务被包装成一个个系统级 service，这些 service 往往会在设备启动之时添加进Android系统。**而 service 实体的底层说到底就是一个 BBinder 实体。**

如果某个程序希望享受系统提供的服务，它就必须调用系统提供的外部接口，向系统发出相应的请求。
因此，Android 中的程序必须先拿到和某个系统 service 对应的代理接口，然后才能通过这个接口，享受系统提供的服务。**说白了就是我们得先拿到一个和目标 service 对应的合法 BpBinder。**
 
**通过 ServiceManager 可以获取 service 对应的代理接口。**

ServiceMananger（SMS） 它的基本任务就是管理其他系统服务。其他系统服务在系统启动之时，就会向 SMS 注册自己，于是 SMS 先记录下与那个 service **对应的名字和句柄值**。有了句柄值就可以用来创建合法的 BpBinder 了。

**句柄** 是个简单的**整数值**，用来告诉 Binder 驱动我们想找的目标 Binder 实体是哪个。**但是请注意，句柄只对发起端进程和 Binder 驱动有意义，A 进程的句柄直接拿到 B 进程，是没什么意义的。也就是说，不同进程中指代相同Binder 实体的句柄值可能是不同的.**
 
而对于 Service Manager Service 这个特殊的服务而言，其对应的代理端的句柄值已经预先定死为 **0** 了，所以我们直接 new BpBinder(0) 拿到的就是个合法的 BpBinder，其对端为 “Service Manager Service实体”

## 具体使用Service Manager Service

要获取某系统 service 的代理接口，必须先**得到 IServiceManager 代理接口**。
【frameworks/native/libs/binder/IServiceManager.cpp】
```c
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) 
        return gDefaultServiceManager;
    {
        AutoMutex _l(gDefaultServiceManagerLock);
        if (gDefaultServiceManager == NULL) 
        {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
        }
    }

    return gDefaultServiceManager;
}
```

主要是：
```c
 gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));

template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}
```

其中 interface_cast 它简单的包装了一下 asInterface 方法。 其中 **getContextObject(NULL) 实际上相当于 new BpBinder(0)** 。也就是说，其实调用的是 **IServiceManager::asInterface(obj)**，而这个 obj 参数就是 new BpBinder(0) 得到的对象。

而在 Java层，是这样获取 IServiceManager 接口的：【frameworks/base/core/java/android/os/ServiceManager.java】
```java
private static IServiceManager getIServiceManager() 
{
    if (sServiceManager != null) {
        return sServiceManager;
    }

    // Find the service manager
    sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
    return sServiceManager;
}
```
也是调用 asInterface 方法，本质上和 C++ 层是一至的。

```java
static public IServiceManager asInterface(IBinder obj){
    if (obj == null) {
        return null;
    }
    
    IServiceManager in = (IServiceManager)obj.queryLocalInterface(descriptor);
    if (in != null)  {
        return in;
    }
    return new ServiceManagerProxy(obj);
}
```

最终会走到 return new ServiceManagerProxy(obj)，也就是说 **ServiceManagerProxy** 就是 IServiceManager代理接口。

**总结：**
 
    用户要访问 Service Manager Service 服务，必须先拿到 IServiceManager 代理接口，而ServiceManagerProxy 就是代理接口的实现。ServiceManagerProxy 的构造函数内部会把 obj 参数记录到mRemote中：

```java
public ServiceManagerProxy(IBinder remote) {
    mRemote = remote;
}
```

mRemote的定义是：

```java
private IBinder mRemote;
```

其实说白了，mRemote 的核心包装的就是句柄为 0 的 BpBinder 对象，这个应该很容易理解。

当我们通过 IServiceManager 代理接口访问 SMS 时，其实调用的就是 ServiceManagerProxy 的成员函数。比如getService()、checkService() 等等。

### 通过 getService() 来获取某系统服务的代理接口

除了注册系统服务，Service Manager Service 的另一个主要工作就是让用户进程可以获取系统 service 的代理接口。
ServiceManagerProxy 中的 getService() 等成员函数，仅仅是把语义整理进 **parcel**，并通过 **mRemote** 将 **parcel** 传递到目标端而已。

```java
public IBinder getService(String name) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IServiceManager.descriptor);
    data.writeString(name);
    
    mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
    
    IBinder binder = reply.readStrongBinder();
    reply.recycle();
    data.recycle();
    return binder;
}
```

传递的语义就是 GET_SERVICE_TRANSACTION，非常简单。mRemote 从本质上看就是句柄为 0 的 BpBinder，所以 binder 驱动很清楚这些语义将去向何方。

# Service Manager Service的主程序
【frameworks\base\cmds\servicemanager\Service_manager.c】

```cpp
int main(int argc, char **argv){
    struct binder_state *bs;
    void *svcmgr = BINDER_SERVICE_MANAGER;

    bs = binder_open(128*1024);

    if (binder_become_context_manager(bs))  {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    svcmgr_handle = svcmgr;
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```

 main() 函数一开始就 **打开binder驱动** ，然后调用 **binder_become_context_manager**() 让自己成为整个系统中唯一的上下文管理器（也就是 service 管理器）。接着 main() 函数调用 **binder_loop**() 进入**无限循环，不断监听并解析 binder 驱动发来的命令。**
 
## binder_open

binder_open() 的作用是**打开 binder 驱动**，驱动文件为 “/dev/binder”

```cpp
struct binder_state * binder_open(unsigned mapsize){
    struct binder_state *bs;

    bs = malloc(sizeof(*bs));
    //. . . . . .
    bs->fd = open("/dev/binder", O_RDWR);
    //. . . . . .
    bs->mapsize = mapsize;
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    //. . . . . .
    return bs;
	//. . . . . .
}
```

**参数 mapsize 表示它希望把 binder 驱动文件的多少字节映射到本地空间。这里是 128*1024 即 128KB。**

Service Manager Service 和普通进程所映射的 binder 大小是不同的，普通进程大小是 **BINDER_VM_SIZE（在ProcessState中定义：#define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)**  
**大小是 1MB-8KB，_SC_PAGE_SIZE 的大小是 4096**
 
# Binder如何精确制导，找到目标Binder实体，并唤醒进程或者线程？
## 创建 binder_proc
当构造 ProcessState 并打开 binder 驱动之时，会调用到驱动层的 binder_open() 函数，而 binder_proc 就是在binder_open() 函数中创建的。新创建的 binder_proc 会作为一个节点，插入一个总链表（binder_procs）中：
【kernel/drivers/staging/android/Binder.c】

```cpp
static int binder_open(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc;
    // . . . . . .
    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
   
    get_task_struct(current);
    proc->tsk = current;
    //. . . . . .
    hlist_add_head(&proc->proc_node, &binder_procs);
    proc->pid = current->group_leader->pid;
    //. . . . . .
    filp->private_data = proc;
    //. . . . . .
}
```
![binder4.png](https://s2.loli.net/2023/06/19/wGeNVZhjAWnmbDk.png)

## binder_proc中的 4 棵红黑树

```cpp
struct binder_proc{
    struct hlist_node proc_node;
    struct rb_root threads;
    struct rb_root nodes;
    struct rb_root refs_by_desc;
    struct rb_root refs_by_node;
    int pid;
    //. . . . . .
};
```

- nodes 树用于记录binder实体
- refs_by_desc 树和 refs_by_node 树则用于记录 binder 代理。之所以会有两个代理树，是为了便于快速查找，我们暂时只关心其中之一就可以了。
- threads 树用于记录执行传输动作的线程信息。

 在一个进程中，有多少“ **被其他进程进行跨进程调用的** ” binder 实体，就会在该进程对应的 **nodes 树**中生成多少个红黑树节点。
另一方面，一个进程要**访问多少其他进程的 binder 实体**，则必须在其 **refs_by_desc 树**中拥有对应的引用节点。

**总结：**  
**比如 进程1 的 BpBinder 在发起跨进程调用时，向 binder 驱动传入了自己记录的句柄值，**
 
**binder 驱动就会在“进程1 对应的 binder_proc 结构 ”的引用树中查找和句柄值相符的 binder_ref 节点，一旦找到 binder_ref 节点，就可以通过该节点的 node 域找到对应的 binder_node 节点，这个目标 binder_node 是从属于进程2 的binder_proc ，**
 
**因为 binder_ref 和 binder_node 都处于 binder 驱动的地址空间中，所以是可以用指针直接指向的。**
 
**目标 binder_node 节点的 cookie 域，记录的其实是进程2 中 BBinder 的地址，binder 驱动只需把这个值反映给应用层，应用层就可以直接拿到 BBinder 了。**
 
**这就是Binder完成精确打击的大体过程。**
 
# binder_proc 为何会有两棵 binder_ref 红黑树？
之所以会有两个代理树，是为了便于快速查找。

# Binder传输数据的大小限制？
**普通用户进程**大小是 **1M-8KB**，在 ProcessState 中定义：

```cpp
#define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)
```
（因为Binder本身就是为了进程间频繁而灵活的通信所设计的，并不是为了拷贝大数据而使用的）

**ServiceManager** 在打开 Binder 驱动时申请了 **128KB** 的大小。（因为 ServcieManager 主要面向系统Service，只是简单的提供一些addServcie，getService的功能，不涉及多大的数据传输，因此不需要申请多大的内存）

内核限制是 4M


# BpBinder 和 IPCThreadState

BpBinder 是代理端的核心，主要负责跨进程传输，并且不关心所传输的内容。  
而 ProcessState 则是进程状态的记录器，它里面记录着打开 binder 驱动后得到的句柄值。

BpBinder如何和ProcessState联系？需要提到IPCThreadState。 

从名字上看，IPCThreadState 是“和跨进程通信（IPC）相关的线程状态”。那么很显然，**一个具有多个线程的进程里应该会有多个 IPCThreadState 对象**，只不过每个线程只需一个 IPCThreadState 对象而已。所以，在实际的代码中，IPCThreadState 对象是存放在线程的局部存储区（TLS）里的。

## BpBinder 的 transact() 动作

```cpp
status_t BpBinder::transact(uint32_t code, const Parcel& data,
Parcel* reply, uint32_t flags){
    // Once a binder has died, it will never come back to life.
    if (mAlive){
        status_t status = IPCThreadState::self()->transact(mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
 
    return DEAD_OBJECT;
}
```

每当我们利用 BpBinder 的 transact() 函数发起一次跨进程事务时，其内部其实是调用 IPCThreadState 对象的transact()。

进程中的一个 BpBinder 有可能被多个线程使用，所以发起传输的 IPCThreadState 对象可能并不是同一个对象，但这没有关系，因为这些 IPCThreadState 对象最终使用的是**同一个 ProcessState 对象**（ 在每个进程中，会有一个全局的 ProcessState 对象 ）

## IPCThreadState 的 transact()

```cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags){
    //. . . . . .
    // 把data数据整理进内部的mOut包中
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    //. . . . . .
   
    if ((flags & TF_ONE_WAY) == 0) {
        //. . . . . .
        if (reply){
            err = waitForResponse(reply);
        }else{
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        //. . . . . .
    }else{
        err = waitForResponse(NULL, NULL);
    }
   
    return err;
}
```

IPCThreadState::transact() 会先调用 writeTransactionData() 函数将 data 数据整理进内部的 mOut 包中，这个函数的代码如下：

```cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
                                              int32_t handle, uint32_t code,
                                              const Parcel& data, status_t* statusBuffer){
    binder_transaction_data tr;
 
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;
   
    //. . . . . .
    tr.data_size = data.ipcDataSize();
    tr.data.ptr.buffer = data.ipcData();
    tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);
    tr.data.ptr.offsets = data.ipcObjects();
    //. . . . . .
   
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));
   
    return NO_ERROR;
}
```

接着 IPCThreadState::transact() 会考虑本次发起的事务是否需要回复。

- “不需要等待回复的”事务，在其 flag 标志中会含有 **TF_ONE_WAY**，**表示一去不回头**。
- 而“需要等待回复的”，则需要在传递时提供记录回复信息的Parcel对象，一般发起 transact() 的用户会提供这个Parcel 对象，如果不提供，transact()函数内部会临时构造一个假的 Parcel 对象。

在 transact 中，实际完成跨进程事务的是 **waitForResponse**() 函数：

```cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult){
    int32_t cmd;
    int32_t err;
 
    while (1) {
        // talkWithDriver()内部会完成跨进程事务
        if ((err = talkWithDriver()) < NO_ERROR)
            break;
       
        // 事务的回复信息被记录在mIn中，所以需要进一步分析这个回复
        . . . . . .
        cmd = mIn.readInt32();
        . . . . . .
        switch (cmd){
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;
       
        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;
 
        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;
        . . . . . .
        . . . . . .
        default:
            // 注意这个executeCommand()噢，它会处理BR_TRANSACTION的。
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }
 
finish:
    . . . . . .
    return err;
}
```

### talkWithDriver
在 waitForResponse 函数中，是通过** talkWithDriver **来和 Binder 驱动打交道的。

```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive){
    . . . . . .
    binder_write_read bwr;
   
    . . . . . .
    bwr.write_size = outAvail;
    bwr.write_buffer = (long unsigned int)mOut.data();
    . . . . . .
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (long unsigned int)mIn.data();
    . . . . . .
    . . . . . .
    do{
        . . . . . .
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        . . . . . .
    } while (err == -EINTR);
 
    . . . . . .
    . . . . . .
    return err;
}
```

说到底会调用 **ioctl()** 函数。因为 ioctl() 函数在传递 **BINDER_WRITE_READ** 语义时，既会使用 “输入buffer”，也会使用 “输出buffer”，所以 IPCThreadState 专门搞了两个 Parcel 类型的成员变量：**mIn** 和 **mOut**。总之就是，  **mOut 中的内容发出去，发送后的回复写进 mIn。**
 
mIn 和 mOut 的 data 会先整理进一个 **binder_write_read** 结构（bwr），然后再传给 ioctl() 函数。

**总结：**

- **在 transact 中，首先把数据整合到 mOut 中，mOut 和 mIn 都是 Parcel 对象，mOut 是发出去，发送后回复写进 mIn。**
- **如果设置了 ONE_WAY 的话，表示“一去不回头” client 端就不等待了，直接回调 binder 里面的 onTransact ，如果不是 ONE_WAY，就需要等待 server 端的处理结果。**
- **如果不是 ONE_WAY，调用 waitForResponse 完成跨进程事务，里面调用 talkWithDriver 来和 Binder 驱动打交道的。**
- **在 talkWithDriver 中，ioctl 函数将数据写到自己的缓存 buffer。**
- **这时候数据在驱动层，还没有传递到 BBinder 一侧 。**

 
## 数据是如何写入红黑树的？

ioctl() 相对的动作是 **binder_ioctl**() 函数。在这个函数里，会先调用类似 **copy_from_user**() 这样的函数，来读取用户态的数据。然后，再调用 **binder_thread_write**() 和 **binder_thread_read**() 进行进一步的处理。

![binder5.png](https://s2.loli.net/2023/06/19/9N4iP5ToOByvmdZ.png)

在调用 binder_thread_write() 之后，binder_ioctl() 接着调用到  binder_thread_read()，**此时往往需要等待远端的回复**，所以 binder_thread_read() 会让线程睡眠，把控制权让出来。
在未来的某个时刻，远端处理完此处发去的语义，就会着手发回回复。当回复到达后，线程会从以前binder_thread_read() 睡眠的地方醒来，并进一步解析收到的回复。

在调用 binder_thread_wirte 方法时，有两个参数，分别是 **binder_proc** 指针和 **binder_thread** 指针，表示发起传输动作的进程和线程，此处的 binder_thread 就是从 **threads** 树中查到的节点。

```cpp
thread = binder_get_thread(proc);
```


在 binder_get_thread 方法中，会尽量从 threads 树中查找和 current 线程匹配的 binder_thread 节点，如果找不到，就会创建一个新的节点并插入树中，所以 threads 树中的节点是在这里创建的。

Binder IPC 机制的大体思路是这样的，它将每次“传输并执行特定语义的”工作理解为一个小事务，系统中当然会有很多事务，那么**发向同一个进程或线程的若干事务**就必须**串行化**起来，因此 binder 驱动为**进程节点（binder_proc）和线程节点（binder_thread）**都设计了个 todo 队列。todo 队列的职责就是“**串行化地组织待处理的事务**”。

事务的类名可以相应地定为 **binder_transaction**

传输动作的基本目标就很明确了，就是想办法把发起端的一个 binder_transaction 节点，插入到目标端进程或其合适子线程的 todo 队列去。

如果binder驱动可以找到一个合适的线程，它就会把 binder_transaction 节点插到它的 todo 队列去。而如果找不到合适的线程，还可以把节点插入目标 **binder_proc** 的 todo 队列。

binder 驱动在传输数据的时候，可不是仅仅简单地递送数据，它会分析被传输的数据，找出其中记录的 binder 对象，并生成相应的树节点。

如果传输的是个 binder 实体对象，它不仅会在发起端对应的** nodes 树**中添加一个 **binder_node** 节点，还会在**目标端**对应的 **refs_by_desc 树、refs_by_node 树**中添加一个 **binder_ref** 节点，而且让 binder_ref 节点的 **node域**指向 binder_node 节点。

![binder6.png](https://s2.loli.net/2023/06/19/zkxiBAET7ShofcK.png)

用红色线条来表示传输 binder 体时在驱动层会添加的红黑树节点以及节点之间的关系。

### 驱动层又是怎么知道所传的数据中有多少binder对象，以及这些对象的确切位置呢？
**是你告诉它的**。在 binder 驱动传递数据之前，都要把数据打成 **parcel** 包。比如：

```cpp
virtual status_t addService(const String16& name, const sp<IBinder>& service){
    Parcel data, reply;
       
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
    data.writeString16(name);
    data.writeStrongBinder(service);  // 把一个binder实体“打扁”并写入parcel
    status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
    return err == NO_ERROR ? reply.readExceptionCode() : err;
}
```

**writeStrongBinder** 方法就是负责把一个binder实体“打扁”并写入 parcel。“打扁”的意思就是把 binder 对象整理成 **flat_binder_object** 变量：

```cpp
status_t Parcel::writeStrongBinder(const sp<IBinder>& val){
    return flatten_binder(ProcessState::self(), val, this);
}

status_t flatten_binder(const sp<ProcessState>& proc, const sp<IBinder>& binder, Parcel* out){
    flat_binder_object obj;
    . . . . . .
    if (binder != NULL) {
        IBinder *local = binder->localBinder();
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();
            . . . . . .
            obj.type = BINDER_TYPE_HANDLE;
            obj.handle = handle;
            obj.cookie = NULL;
        } else {
            obj.type = BINDER_TYPE_BINDER;
            obj.binder = local->getWeakRefs();
            obj.cookie = local;
        }
    }
    . . . . . .
    return finish_flatten_binder(binder, obj, out);
}
```

- flat_binder_object 用 cookie 域记录 binder 实体的指针，即BBinder指针。
- 如果打扁的是 binder 代理，那么 flat_binder_object 用 handle 域记录的 binder 代理的句柄值。

最后调用了 **finish_flatten_binder** 函数，这个函数内部会记录下刚刚被扁平化的 flat_binder_object 在 parcel 中的位置。
![binder7.png](https://s2.loli.net/2023/06/19/dpQB2WZg3Dmhv7o.png)

在 parcel 中，内部会有一个buffer，记录着 parcel 中所有扁平化的数据，有些扁平数据是普通数据，有些是 binder 对象。所以 parcel 中会构造一个 mObjects 数组，专门记录那些 binder 扁平数据所在的位置。

上面说到，在 transact 函数中的 writeTransactionData 函数中，将 data 数据整理进内部的 mOut 包：

```cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
                                              int32_t handle, uint32_t code,
                                              const Parcel& data, status_t* statusBuffer){
    binder_transaction_data tr;
    . . . . . .
    // 这部分是待传递数据
    tr.data_size = data.ipcDataSize();
    tr.data.ptr.buffer = data.ipcData();
    // 这部分是扁平化的binder对象在数据中的具体位置
    tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);
    tr.data.ptr.offsets = data.ipcObjects();
    . . . . . .
    mOut.write(&tr, sizeof(tr));
    . . . . . .
}
```

其中给 tr.data.ptr.offsets 赋值的那句，即所做的就是记录下“待传数据”中所有binder对象的具体位置，示意图如下：

![binder8.png](https://s2.loli.net/2023/06/19/hPTswMFQaUYvAHO.png)

所以，当 **binder_transaction_data** 传递到 binder 驱动层后，驱动层可以准确地分析出数据中到底有多少 binder对象，并分别进行处理，从而产生出合适的红黑树节点。

**此时，如果产生的红黑树节点是 binder_node 的话，binder_node 的 cookie 域会被赋值成 flat_binder_object 所携带的 cookie 值 ，也就是用户态的 BBinder 地址值。**

**这个新生成的 binder_node 节点被插入红黑树后，会一直严阵以待，以后当它成为另外某次传输动作的目标节点时，它的 cookie 域就派上用场了，此时 cookie 值会被反映到用户态，于是用户态就拿到了 BBinder 对象。**
 
**再具体看一下 IPCThreadState::waitForResponse() 函数，当它辗转从睡眠态跳出来时，会进一步解析刚收到的命令，此时会调用** ecuteCommand(cmd) ： 

```cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult){
    int32_t cmd;
    int32_t err;
 
    while (1) {
        if ((err = talkWithDriver()) < NO_ERROR) break;
        //. . . . . .
        switch (cmd){
            //. . . . . .
        default:
            err = executeCommand(cmd);
            //. . . . . .
            break;
        }
    }
    //. . . . . .
    return err;
}

status_t IPCThreadState::executeCommand(int32_t cmd){
    BBinder* obj;
    //. . . . . .
    switch (cmd){
        //. . . . . .
        case BR_TRANSACTION:{
            binder_transaction_data tr;
            result = mIn.read(&tr, sizeof(tr));
            //. . . . . .
            if (tr.target.ptr){
                sp<BBinder> b((BBinder*)tr.cookie);
                const status_t error = b->transact(tr.code, buffer, &reply, tr.flags);
                if (error < NO_ERROR) reply.setError(error);

            }
            //. . . . . .
            if ((tr.flags & TF_ONE_WAY) == 0){
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                sendReply(reply, 0);
            }else{
                LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
            }
            //. . . . . .
        }
        break;
        //. . . . . .
        default:
            printf("*** BAD COMMAND %d received from Binder driver\n", cmd);
            result = UNKNOWN_ERROR;
            break;
    }
    //. . . . . .
    return result;
}
```

注意上面代码中的 **sp<BBinder> b((BBinder*)tr.cookie)** 一句，驱动层的 binder_node 节点的 cookie 值终于发挥它的作用了，我们拿到了一个合法的 **sp<BBinder>**。

接下来，程序走到 **b->transact()** 一句。transact() 函数的代码截选如下：

```cpp
status_t BBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags){
    . . . . . .
    switch (code) {
        . . . . . .
        default:
            err = onTransact(code, data, reply, flags);
            break;
    }
    . . . . . .
}
```

其中最关键的一句是调用 **onTransact**()。因为我们的 binder 实体在本质上都是继承于 BBinder 的，而且我们一般都会重载 onTransact() 函数，所以上面这句 onTransact() 实际上调用的是具体 binder 实体的 onTransact() 成员函数。


# 如果我client端发了两个线程去调用server的binder服务 server是怎么执行的？是单线程执行还是多线程执行？是并行还是串行执行？

无论 client 用多少个线程调用 server，server 都是一个 binder 去处理。

binder 的进程与线程关系：

![binder9.png](https://s2.loli.net/2023/06/19/gEq8pBYyRxO2HFv.png)

**用户空间：**ProcessState 描述一个进程，IPCThreadState 描述一个进程中对应的一个线程  
**内核空间：**binder_proc 描述一个进程，统一由 binder_procs 全局连表保存，binder_thread 对应进程的一个线程  
ProcessState 与 binder_proc 一一对应。

在 ServiceManager addService 时，service 在 new 的时候会启动线程池，并把当前线程加到线程池中，例如：

```cpp
int main(int argc __unused, char** argv)
{
    ...
    //启动Binder线程池
    ProcessState::self()->startThreadPool();
    //当前线程加入到线程池
    IPCThreadState::self()->joinThreadPool();
 }
```

## Binder 的主进程是在哪创建的？
我们知道，App 的进程都是咋 zygote 中 fork 出来的。在 fork 出一个子进程的时候，会调用 **AppRuntime.onZygoteInit()**

```cpp
virtual void onZygoteInit(){
    sp<ProcessState> proc = ProcessState::self();
    ALOGV("App process: starting thread pool.\n");
    proc->startThreadPool();
}
```

里面调用 ProcessState 的 startThreadPool 方法:


```cpp
void ProcessState::startThreadPool(){
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);   //注意这里传递的参数是 true
    }
}

void ProcessState::spawnPooledThread(bool isMain){
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
```

isMain 参数表示的就是 **是否主线程，**PoolThread 的定义如下： 

```cpp
class PoolThread : public Thread{
public:
    explicit PoolThread(bool isMain) : mIsMain(isMain)  {
    }

protected:
    virtual bool threadLoop(){
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }

    const bool mIsMain;
};

```

对于一个 Server 进程 Binder 线程数限制为 **15** 个，在 ProcessState 中定义： **DEFAULT_MAX_BINDER_THREADS = 15**
 
看上面的 joinThreadPool 代码：

```cpp
void IPCThreadState::joinThreadPool(bool isMain)
{
    ////创建Binder线程
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand(); // //处理下一条指令

        //非主线程出现timeout则线程退出
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);
 
    mOut.writeInt32(BC_EXIT_LOOPER);  //线程退出循环
    talkWithDriver(false); //false代表bwr数据的read_buffer为空
} 
```

- 对于isMain=true 的情况下，command为 **BC_ENTER_LOOPER**，代表的是 Binder 主线程，不会退出的线程；
- 对于isMain=false 的情况下，command为 **BC_REGISTER_LOOPER**，表示是由 binder 驱动创建的线程。

**上面代码可以看出，只有主线程会阻塞循环等待，非主线程则跳出循环，所以实际处理任务的是非主线程**

joinThreadPool 中开启了一个死循环监听，然后调用 getAndExecuteCommand：

![binder10.png](https://s2.loli.net/2023/06/19/AKCDd9X73bankBp.png)

在 getAndExecuteCommand 中，调用 talkWithDriver 与 Binder 驱动打交道，还有 **executeCommand **函数。如果需要回复会进一步调用 sendReply 函数。

**所以这个问题的答案是：**
 
**单线程发起请求的情况下，binder端是多线程串行执行的。**  
**多线程发起请求的情况下，binder端是多线程并行执行的。**  
**binder端有binder线程池去执行。**  
**每个进程有一个todo队列，每个线程里面也有一个todo队列。**  
