# Binder 线程总结

Binder 设计架构中，只有第一个 Binder 主线程(也就是Binder_1线程)是由应用程序主动创建，Binder 线程池的普通线程都是由Binder驱动根据IPC通信需求创建，Binder线程的创建流程图：
![binder_thread_create.jpg](https://cdn.nlark.com/yuque/0/2020/jpeg/450005/1584061970066-cb186fb9-77a7-4896-bae9-c3092e28993e.jpeg#align=left&display=inline&height=895&originHeight=895&originWidth=972&size=62914&status=done&style=none&width=972)

每次由 Zygote fork 出新进程的过程中，伴随着创建 binder 线程池，调用 spawnPooledThread 来创建 binder 主线程。**当线程执行 binder_thread_read 的过程中，发现当前没有空闲线程，没有请求创建线程，且没有达到上限，则创建新的 binder 线程。**

# Binder 一次拷贝原理

**内存映射**
由于应用程序不能直接操作设备硬件地址，所以操作系统提供了一种机制：内存映射，把设备地址映射到进程虚拟内存区。
举个例子，如果用户空间需要读取磁盘的文件，如果不采用内存映射，那么就需要在内核空间建立一个页缓存，页缓存去拷贝磁盘上的文件，然后用户空间拷贝页缓存的文件，这就需要两次拷贝。
采用内存映射，如下图所示。
[![](https://cdn.nlark.com/yuque/0/2020/png/450005/1586394068889-1e9ab78d-7367-41eb-b16c-7c4253249995.png#align=left&display=inline&height=367&originHeight=367&originWidth=835&size=0&status=done&style=none&width=835)](https://s2.ax1x.com/2019/09/21/nzlnaV.png)nzlnaV.png
由于新建了虚拟内存区域，那么磁盘文件和虚拟内存区域就可以直接映射，少了一次拷贝。
内存映射全名为Memory Map，在Linux中**通过系统调用函数mmap来实现内存映射。将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间，反之亦然。内存映射能减少数据拷贝次数，实现用户空间和内核空间的高效互动**


Android 选择 Binder 作为主要进程通信的方式同其性能高也有关系，

Binder 只需要一次拷贝就能将 A 进程用户空间的数据为 B 进程所用。这里主要涉及两个点：

- Binder 的 map 函数，会将**内核空间**直接与**用户空间**对应，**用户空间可以直接访问内核空间的数据**
- A 进程的数据会被直接拷贝到 B 进程的内核空间（一次拷贝）

```cpp
 #define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))

ProcessState::ProcessState()
    : mDriverFD(open_driver())
        , mVMStart(MAP_FAILED)
        , mManagesContexts(false)
        , mBinderContextCheckFunc(NULL)
        , mBinderContextUserData(NULL)
        , mThreadPoolStarted(false)
        , mThreadPoolSeq(1){
        if (mDriverFD >= 0) {
            ....
            // mmap the binder, providing a chunk of virtual address space to receive transactions.
            mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
            ...
        }
}
```

mmap 函数会调用到 binder 驱动中的 **binder_mmap** 函数，
主要功能：

- 首先在内核虚拟地址空间，申请一块与用户虚拟内存相同大小的内存；
- 然后再申请 1 个 page 大小的物理内存，
- 再将**同一块物理内存**分别映射到**内核虚拟地址空间**和**用户虚拟内存空间**，**从而实现了用户空间的 Buffer 和内核空间的 Buffer 同步操作的功能。**

```cpp
static int binder_mmap(struct file *filp, struct vm_area_struct *vma){
    int ret;
    struct vm_struct *area; //内核虚拟空间
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;
    struct binder_buffer *buffer;  //【见附录3.9】

    if (proc->tsk != current)
        return -EINVAL;

    if ((vma->vm_end - vma->vm_start) > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;  //保证映射内存大小不超过4M

    mutex_lock(&binder_mmap_lock);  //同步锁
    //采用IOREMAP方式，分配一个连续的内核虚拟空间，与进程虚拟空间大小一致
    area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
    if (area == NULL) {
        ret = -ENOMEM;
        failure_string = "get_vm_area";
        goto err_get_vm_area_failed;
    }
    proc->buffer = area->addr; //指向内核虚拟空间的地址
    //地址偏移量 = 用户虚拟地址空间 - 内核虚拟地址空间
    proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
    mutex_unlock(&binder_mmap_lock); //释放锁

    ...
    //分配物理页的指针数组，数组大小为vma的等效page个数；
    proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
    if (proc->pages == NULL) {
        ret = -ENOMEM;
        failure_string = "alloc page array";
        goto err_alloc_pages_failed;
    }
    proc->buffer_size = vma->vm_end - vma->vm_start;

    vma->vm_ops = &binder_vm_ops;
    vma->vm_private_data = proc;

    //分配物理页面，同时映射到内核空间和进程空间，先分配1个物理页  
    if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {
        ret = -ENOMEM;
        failure_string = "alloc small buf";
        goto err_alloc_small_buf_failed;
    }
    buffer = proc->buffer; //binder_buffer对象 指向proc的buffer地址
    INIT_LIST_HEAD(&proc->buffers); //创建进程的buffers链表头
    list_add(&buffer->entry, &proc->buffers); //将binder_buffer地址 加入到所属进程的buffers队列
    buffer->free = 1;
    //将空闲buffer放入proc->free_buffers中
    binder_insert_free_buffer(proc, buffer);
    //异步可用空间大小为buffer总大小的一半。
    proc->free_async_space = proc->buffer_size / 2;
    barrier();
    proc->files = get_files_struct(current);
    proc->vma = vma;
    proc->vma_vm_mm = vma->vm_mm;
    return 0;

    ...// 错误flags跳转处，free释放内存之类的操作
    return ret;
}
```

binder_mmap 通过加锁，保证一次只有一个进程分配内存，保证多进程间的并发访问。

**当数据从用户空间拷贝到内核空间的时候，是直从当前进程的用户空间接拷贝到目标进程的内核空间，这个过程是在请求端线程中处理的，操作对象是目标进程的内核空间**
**
**![11.jpg](https://cdn.nlark.com/yuque/0/2020/jpeg/450005/1584063509803-baa2868c-b429-4fdc-96ca-4f17d95d75a6.jpeg#align=left&display=inline&height=817&originHeight=817&originWidth=1240&size=67821&status=done&style=none&width=1240)**
**
# 系统服务与bindService等启动的服务的区别?
系统服务一般是在系统启动的时候，由 **SystemServer** 进程创建并注册到 **ServiceManager** 中的。
而普通服务一般是通过 **ActivityManagerService** 启动的服务，或者说通过四大组件中的 Service 组件启动的服务。

这两种服务在实现跟使用上是有不同的，主要从以下几个方面：

- 服务的启动方式
- 服务的注册与管理
- 服务的请求使用方式

## 服务的启动方式

**系统服务**一般都是 **SystemServer 进程负责启动**，比如 S，WMS，PKMS，电源管理等，这些服务本身其实实现了Binder接口，作为 Binder 实体注册到 ServiceManager 中，**被 ServiceManager 管理。**
而 SystemServer 进程里面会启动一些 Binder 线程，主要用于监听 Client 的请求，并分发给响应的服务实体类，可以看出，这些系统服务是位于SystemServer进程中。

**bindService 类型的服务**，这类服务一般是通过 Activity 的 startService 或者其他 context 的 startService 启动的，这里的 Service 件只是个封装，主要的是里面 Binder 服务实体类。
这个启动过程不是 ServcieManager 管理的，而是通过 **ActivityManagerService** 进行管理的，同 Activity 管理类似。
 
## 服务的注册与管理

**系统服务**一般都是通过 **ServiceManager** 的 **addService** 进行注册的，这些服务一般都是需要拥有特定的权限才能注册到 ServiceManager。

**bindService 启动的服务**可以算是注册到 **ActivityManagerService**，只不过 ActivityManagerService 管理服务的方式同 ServiceManager 不一样，**而是采用了 Activity 的管理模型。**
**
## 使用方式

**系统服务**一般都是通过 **ServiceManager** 的 **getService** 得到服务的**句柄**，这个过程其实就是去 ServiceManager 中查询注册系统服务。

**bindService 启动的服务**，主要是去 ActivityManagerService 中去查找相应的 Service 组件，最终会将 Service 内部 Binder 的句柄传给 Client。

![322.jpg](https://cdn.nlark.com/yuque/0/2020/jpeg/450005/1584064260519-31fcb855-2604-4db8-a3a4-161940228da1.jpeg#align=left&display=inline&height=1115&originHeight=1115&originWidth=1240&size=115581&status=done&style=none&width=1240)


# 整理回顾

1. **Binder 可理解为一种跨进程的通信手段和一种远程调用手段。可分为代理端和响应端 -> BpBinder 和 BBinder。如果要想实现远程调用，还需要代理接口和接口实现，分别对应 BpInterface 和 BnInterfact, BpBinder 被聚合到 BpInterface 中，BnInterface 继承与 BBinder。**

2. **service 本质上来说是一个 BBinder，在设备启动时，系统服务会在 ServiceManager 中注册自己，ServiceManager 会记录他们的名字和句柄，句柄是一个整数值。要拿到对应的服务，就需要先拿到对应的服务的代理接口，然后通过代理接口，才能拿到对应服务。也就是说要拿到一个合法的 BpBinder**

3. **ServiceManager 的句柄值写死为 0 ，所以通过 new BpBinder(0) 即可拿到 ServiceManager 的代理接口。**

4. **ProcessState 是进程状态，一个进程只有一个全局的 ProcessState，里面记录着打开binder驱动后得到的句柄值。**

5. **在 ServiceManager 主函数中，首先会通过 binder_open 函数打开 binder 驱动，同时申请了 128KB 的内存空间。**

6. **ProcessState 是一个单利模式，在构造时，会调用 binder_open 函数打开对应进程的 binder 驱动，同时指定默认最大线程数为 15.**

7. **ProcessState 的 binder_open 函数里面，会创建一个 binder_proc，一个 binder_proc 可以理解为一个进程，创建的 binder_proc 会作为一个节点加入到一个总链表 binder_procs 中。**

8. **在 binder_proc 中，有四颗红黑树，分别是 nodes，threads，ref_by_desc，ref_by_node。**

9. **nodes 红黑树用于记录 binder 实体**

10. **ref_by_desc 和 ref_by_node 红黑树用于记录 binder 代理。**

11. **threads 红黑树用于记录传输过程中的线程信息**

12. **BpBinder 是代理端核心，负责跨进程传输，但不关心传输的内容，ProcessState 负责记录进程状态，BpBinder 要联系 ProcessState，则需要 IPCThreadState，从名字上看，它是负责记录 跨进程线程状态的类。**

13. **一个具有多线程的进程会有多个 IPCThreadState,IPCThreadState 在线程内是单例的。**

14. **BpBinder 的 transact 方法负责发起一次跨进程事务，里面实际调用的是 IPCThreadState 的 transtact 方法。**

15. **可得，当 BpBinder 被多线程使用时，发起的事务可能不是来自同一个 IPCThreadState 对象，但是没关系，因为他们所对应的 ProcessState 对象是同一个。**

16. **在 IPCThreadState 的 transtact 方法中，它会先把传输数据整理到一个 mOut 的 Parcel 对象中，然后判断是不是 oneway（TF_ONE_WAY），如果是 oneway，则代表一去不复返，则 client 不需要等待 service 的回复，如果不是 oneway 的话，就需要等待回复。**

17. **无论需不需要回复，他们都会调用 waitForResponse 方法来完成跨进程事务，waitForResponse 会传入是否要回复的参数。**

18. **waitForResponse 里面是一个 while 死循环，里面会调用 talkWithDriver 来完成具体的跨进程事务工作，talkWithDriver 就是用来和 Binder 驱动打交道的。**

19. **在 talkWithDriver 里面，实际上会调用 ioctl 方法，在 ioctl 方法中，会传入 BINDER_WRITE_READ 语义，然后把缓存数据写入。**

20. **在 ioctl 函数中，会调用内核层的 binder_ioctl，binder_ioctl 中，会接着调用 binder_thread_write() 和 binder_thread_read() 。在调用 binder_thread_read 时，往往需要等待远端回复，所以 binder_thread_read 会让线程睡眠，等远端处理完此处发出的语意时，回复到达后，才会醒来接收和解析回复。**

21. **在 binder_thread_write 方法中，有两个参数，分别是 binder_proc 指针 和 binder_thread 指针，分别表示传输动作的进程和线程，binder_thread 是在 threads 红黑树中查找的。通过 binder_get_thread(proc) 方法**

22. **在 binder_get_thread 方法中，他的逻辑是会尽量在 thread 树中查找出和当前线程匹配的 binder_thread 节点，如果找不到，就会创建一个新节点插入树中，所以 threads 树节点是在这里创建的。**

23. **binder 的 IPC 思路大体是：将每次传输并执行特点语意的工作理解为一个小事务，如果向同一个进程或线程发起多个事务，就必须串行化起来，因为 binder 驱动为进程节点 binder_proc 和线程节点 binder_thread 都设计了 todo 队列，todo 队列的职责就是串行化的组织待处理的事务。**

24. **事务的类名可定义为 binder_transaction**

25. **所以传输动作的基本目标就是吧发起端的一个 binder_transaction 节点插入到目标端进程或其合适子线程的 todo 队列去。**

26. **binder 在传输数据时，如果传输的是个 binder 对象，它不仅会在发起端对应的 nodes 树中插入一个 binder_node 节点，还会在目标端对应的 refs_by_desc 和 refs_by_node 树中插入一个 binder_ref 节点。而且会让 binder_ref 节点的 node 域指向 binder_node 节点。**

27. ** binder 是如果做到精确查找到目标 binder 实体的，因为在 BpBinder 发起跨进程调用时，会向 binder 驱动中传入并且自己的句柄值，binder 驱动会在 进程A 对应的 binder_proc 的引用红黑树中查找和句柄值相符合的 binder_ref 节点，找到后就会通过该节点的 node 域在 nodes 树中找到对应的 binder_node 节点，这个目标 binder_proc 是属于 进程B 的，目标 binder_node 节点的 cookie 域，记录的其实是 进程B 中的 BBinder 地址，binder 驱动只需要把这个地址反映给应用层，就能直接拿到 BBinder 了。**

28. **binder 如何知道在传输的数据中有多少个 binder 对象，以及这些对象的确切位置？因为在数据传输前，都要把数据打成 Parcel 包，Parcel 的 wirteStrongBinder 方法就是负责把一个 binder 实体打扁并写入 Parcel 的。在这个过程中，其实就是把数据整理成 flat_binder_object 对象，其中该对象用 cookie 域记录 binder 实体的指针，即 BBinder 指针，如果打扁的 binder 代理，用 handle 代理记录 binder 代理的句柄值**

29. **整理完 flat_binder_object 对象后，在调用 finish_flatten_binder 函数记录 flat_binder_object 对象在 Parcel 中的位置。**

30. **得到对象和位置后，在 IPCThreadState 的 transact 函数里，把数据整理成 mOut 时，就会记录到了 mOut 中最后插入红黑树。**

31. **在之前说的 waitForResponse 中，当与 binder 驱动打完交道后，会跳出来解析收到的 cmd 命令，这时候调用到了 executeCommand , 在里面主要处理的是 BR_TRANSACTION 语意，这时候会通过 cookie 域拿到 BBinder ，然后调用它的 transact 函数（实际上调用的是 onTransact）,如果需要回复，还会调用 sendReply 函数**

32. **binder 的主线程是在进程被 fork 出来的时候，通过 AppRuntime.onZygoteInite 方法调用 ProcessState 的 startThreadPool 方法创建线程池，并通过 joinThreadPool 添加一个主线程**

33. **在 joinThreadPool 中，会有一个无限死循环来不断的监听和处理指令，通过 getAndExecuteCommand 方法，注意是主线程在做循环，非主线程去处理指令，如果非主线程等待超时则退出循环。**

34. **在 getAndExecuteCommand 方法中，其实也是通过 talkWithDriver 方法与 binder 驱动交互，并且通过 executeCommand 方法调用 BBinder。**




















































