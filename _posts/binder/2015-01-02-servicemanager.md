---
layout: post
title:  "5.2 ServiceManager管家"
date:   2014-01-02 20:30:00
catalog:  true
---

## 5.2 ServiceManager管家

### 5.2.1 启动ServiceManager服务
Android系统对外提供了非常丰富的服务功能， 例如Java层的ActivityManagerService, WindowManagerService服务， Native层的SurfaceFlinger, AudioFlinger服务等，这么多服务有有一个统一的地方来管理这些服务ServiceManager。当系统进程需要增加一个服务时，只需要将服务名和服务实体告诉ServiceManager就可以完成，这便是服务注册过程；当应用进程需要使用某个服务，只需要将服务名告诉ServiceManager就可以查询到服务的代理对象，这便是服务查询过程。

如图5-1所示， AMS注册过程就是告诉ServiceManager进程，ActivityManagerService服务实体运行在system_server进程，服务名叫“activity”，则在servicemanager进程的svclist列表中增加一条svcinfo记录， 里面主要记录着服务名以及相对应的handle值。查询AMS服务的过程，向ServiceManager进程查询一个服务名为“activity”的服务，ServiceManager通过检索svclist列表会找到所对应的服务在该进程中的handle值，有了这个handle值，经过Binder驱动就能生成AMS服务实体的代理对象，有了代理对象就可以使用AMS服务，比如startService。


![ServiceManager进程](/images/book/binder/5-2-1-service_manager.jpg)

整个Binder系统中最先启动的是ServiceManager进程，ServiceManager作为Binder IPC的大管家，统管所有的Binder服务信息。 ServiceManager本身是一个Binder服务， 该服务是任何应用都可以找到的，该服务所对应的handle值为0。Binder框架本身提供一套多线程模型来与Binder驱动通信。ServiceManager进程并没有使用libbinder框架代码，而是自行编写了binder.c直接和Binder驱动来通信，ServiceManager是单线程的进程， 不断地循环在binder_loop()过程来读取和处理事务，从而对外提供查询和注册服务的功能，这样的好处是简单而高效。


**解析servicemanager.rc**

ServiceManager是由init进程通过解析servicemanager.rc文件而创建的，如下：

```
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    writepid /dev/cpuset/system-background/tasks
```

init进程解析后，找到其所对应的可执行程序/system/bin/servicemanager，先来从servicemanager的启动过程说起。

    // service_manager.c
    int main(int argc, char **argv)
    {
        struct binder_state *bs;
        char *driver;
        if (argc > 1) {
            driver = argv[1];
        } else {
            driver = "/dev/binder"; // 默认的Binder设备节点
        }

        //Step 1: 打开binder驱动，申请128k字节内存
        bs = binder_open(driver, 128*1024);
        ...

        //Step 2: 成为上下文管理者
        if (binder_become_context_manager(bs)) {
            return -1;
        }
        ...

        //Step 3: 进入无限循环，处理client端发来的请求
        binder_loop(bs, svcmgr_handler);
        return 0;
    }

启动过程主要划分为以下几个阶段：

- 首先，调用binder_open()方法来打开binder驱动，默认地采用/dev/binder设备节点，申请地内存空间大小为128KB；
- 其次，调用binder_become_context_manager()方法，将自己注册成为binder服务的大管家；
- 最后，调用binder_loop()方法进入无限循环，作为守护进程，随时待命等待处理client端发来的请求。

图5-2所示 描述了这一过程。

![create_servicemanager](/images/binder/create_servicemanager/create_servicemanager.jpg)


#### 1. 打开设备驱动

    // servicemanager/binder.c
    struct binder_state *binder_open(const char* driver, size_t mapsize)
    {
        struct binder_state *bs;
        bs = malloc(sizeof(*bs));

        // Step 1: 打开Binder设备驱动
        bs->fd = open(driver, O_RDWR);

        // Step 2: 验证binder版本是否一致
        if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
            (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
            goto fail_open;
        }

        // Step 3: 通过系统调用，mmap内存映射，mmap必须是page的整数倍
        bs->mapsize = mapsize;
        bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
        return bs;

    fail_map:
        close(bs->fd);
    fail_open:
        free(bs);
        return NULL;
    }

> binder_open的过程，不管是malloc，open，mmap 一旦出现失败都是执行相应清理动作后直接退出， 比如结束，对于这些异常流程为了精简篇幅， 一般情况都会直接省略。

Linux设备驱动可以简单理解成一个文件， 调用open()以可读写方式打开/dev/binder设备节点， 经过系统调用陷入内核，然后由文件系统层层转换，最终进入Binder驱动会执行binder_open方法。binder_open主要工作是在Binder驱动层创建结构体binder_proc，再将binder_proc保存到fd->private_data，同时放入全局链表binder_procs。

通过ioctl系统调用向Binder驱动发送命令BINDER_VERSION，进入Binder驱动根据该命令则获取驱动层定义的BINDER_CURRENT_PROTOCOL_VERSION，
跟用户层同名变量进行比对， 看看binder协议版本是否一致， 目前版本只用于区别是32位或64位。

调用mmap()进行内存映射，映射的内存大小为128KB, mmap()方法经过系统调用对应于Binder驱动层的binder_mmap()，该方法主要工作是在Binder驱动层创建Binder_buffer对象，并放入当前binder_proc结构体的链表proc->buffers。

打开Binder设备驱动过程中的信息都记录在binder_state结构体的成员变量：

- fd：记录进程打开的dev/binder的文件描述符；
- mapped：记录mmap的内存地址；
- mapsize：记录mmap所对应的内存大小；

> binder_open, binder_mmap, binder_ioctl都是Binder驱动层的知识点， 很多书籍一开始就深入Binder驱动层，理解起来会比较难， 不太适合阅读过程，这里先让读者有一定概念，后续小节再进一步展开讲解。

#### 2. 注册成为大管家

    // servicemanager/binder.c
    int binder_become_context_manager(struct binder_state *bs)
    {
        return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
    }

通过ioctl系统调用向Binder驱动发送命令BINDER_SET_CONTEXT_MGR，成为上下文的管理者，由于servicemanager进程启动非常早，可以确定在Binder整体机制正式投入产线之前，就能完成向Binder驱动注册成为大管家的工作。 关于驱动层处理BINDER_SET_CONTEXT_MGR命令的主要任务：

- 保证每个Binder上下文有且仅有一个binder管家实体，如果已存在则不再创建
- 创建binder管家实体，初始化异步事务和binder工作两个队列，并分别增加其强弱引用计数
- 初始化当前binder_context的管家实体（binder_context_mgr_node）和管家uid(binder_context_mgr_uid)信息
- handle等于0的服务实体都是指servicemanager管家实体

> binder_context是Android 8.0新引入的结构体，用于支持Project Treble， 提供多个Binder域(上下文)，包括/dev/binder, /dev/hwbinder，/dev/vndbinder，具体每个Binder域含义后续再讲解。


#### 3. 等待客户请求

    // servicemanager/binder.c
    void binder_loop(struct binder_state *bs, binder_handler func)
    {
        int res;
        struct binder_write_read bwr;
        uint32_t readbuf[32];

        bwr.write_size = 0;
        bwr.write_consumed = 0;
        bwr.write_buffer = 0;

        readbuf[0] = BC_ENTER_LOOPER;
        //向binder驱动发送BC_ENTER_LOOPER协议
        binder_write(bs, readbuf, sizeof(uint32_t));

        for (;;) {
            bwr.read_size = sizeof(readbuf);
            bwr.read_consumed = 0;
            bwr.read_buffer = (uintptr_t) readbuf;
            //等待客户的数据
            res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
            //解析binder信息
            res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        }
    }

servicemanager先向Binder驱动发送BC_ENTER_LOOPER协议，让ServiceManager进入循环。然后再向驱动发送BINDER_WRITE_READ命令， 进入内核态，等待客户端的请求数据。若没有数据，则进入等待状态，直到收到数据后返回用户态，解析并处理，周而复始地不断循环该过程。

（1）先来看看ServiceManager进入循环前的动作

    // servicemanager/binder.c
    int binder_write(struct binder_state *bs, void *data, size_t len)
    {
        struct binder_write_read bwr;
        int res;
        bwr.write_size = len; //此时len = 4
        bwr.write_consumed = 0;
        bwr.write_buffer = (uintptr_t) data; //此时data为BC_ENTER_LOOPER
        bwr.read_size = 0;
        bwr.read_consumed = 0;
        bwr.read_buffer = 0;
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        return res;
    }

根据传递进来的参数初始化bwr，其中write_size大小为4，write_buffer指向缓冲区的起始地址，其内容为BC_ENTER_LOOPER请求协议号。通过ioctl将bwr数据发送给binder驱动。驱动收到BC_ENTER_LOOPER协议，主要工作是将servicemanager的当前线程的looper状态为BINDER_LOOPER_STATE_ENTERED(代表Binder主线程进入循环状态)。

（2） 再看看servicemanager解析过程

通过ioctl向Binder驱动发送BINDER_WRITE_READ命令，这是向驱动进行读与写操作的命令，当写缓存有数据则会执行写操作， 当读缓存有数据则会执行读操作。 如果读缓存没有没有数据时，则等待客户端发起请求， servicemanager一旦读取到事件，则会把数据保存到readbuf，然后交由binder_parse来解析。

    // servicemanager/binder.c
    int binder_parse(struct binder_state *bs, struct binder_io *bio,
                     uintptr_t ptr, size_t size, binder_handler func)
    {
        int r = 1;
        uintptr_t end = ptr + (uintptr_t) size;
        while (ptr < end) {
            uint32_t cmd = *(uint32_t *) ptr; //指向readbuf
            ptr += sizeof(uint32_t);
            switch(cmd) {
            case BR_TRANSACTION: {
                struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
                if (func) {                   // 是指svcmgr_handler()函数
                    unsigned rdata[256/4];
                    struct binder_io msg;
                    struct binder_io reply;
                    int res;
                    bio_init(&reply, rdata, sizeof(rdata), 4); // 初始化binder_io结构体
                    bio_init_from_txn(&msg, txn); //将binder_transaction_data信息解析成binder_io
                    res = func(bs, txn, &msg, &reply);  // 真正处理业务逻辑的核心地方
                    if (txn->flags & TF_ONE_WAY) {
                        binder_free_buffer(bs, txn->data.ptr.buffer);
                    } else {
                        binder_send_reply(bs, &reply, txn->data.ptr.buffer, res); //发送应答数据
                    }
                }
                ptr += sizeof(*txn);
                break;
            }
            case BR_DEAD_BINDER: {
                struct binder_death *death = (struct binder_death *)(uintptr_t) *(binder_uintptr_t *)ptr;
                ptr += sizeof(binder_uintptr_t);
                death->func(bs, death->ptr); //处理binder死亡消息
                break;
            }
            case ...
            default:
                return -1;
            }
        }
        return r;
    }

在binder_parse过程主要工作就是针对不同的BR协议，采用不同的行动，这里涉及到的BR协议有以下4类:

- 第一类：BR_NOOP，BR_TRANSACTION_COMPLETE，BR_INCREFS，BR_ACQUIRE，BR_RELEASE，BR_DECREFS， 这些协议对于ServiceManager进程不做什么操作；
- 第二类：BR_TRANSACTION，BR_REPLY，  接收到binder事务，这是最常用的协议
- 第二类：BR_DEAD_BINDER， 对端binder服务所在进程死亡后的通知处理
- 第四类：BR_FAILED_REPLY，BR_DEAD_REPLY，这些协议代表Binder通信过程出现异常

当收到BR_TRANSACTION协议，则对外提供服务的功能；ServiceManager对外提供的框架代码中所有的功能都是同步的Binder调用，在执行完注册或查询服务后会再发送应答数据

#### 4. 对外提供服务

ServiceManager对外提供查询/注册功能， 便是通过接收到客户端进程发送过来的BR_TRANSACTION协议。接下来， 看看ServiceManager大管家的核心工作内容。

    // service_manager.c
    int svcmgr_handler(struct binder_state *bs,
                       struct binder_transaction_data *txn,
                       struct binder_io *msg,
                       struct binder_io *reply)
    {
        struct svcinfo *si;
        uint16_t *s;
        size_t len;
        uint32_t handle;
        uint32_t strict_policy;
        int allow_isolated;
        ...
        strict_policy = bio_get_uint32(msg);
        s = bio_get_string16(msg, &len);
        ...

        switch(txn->code) {
        case SVC_MGR_GET_SERVICE:
        case SVC_MGR_CHECK_SERVICE:
            s = bio_get_string16(msg, &len); //服务名
            //根据名称查找相应服务
            handle = do_find_service(bs, s, len, txn->sender_euid, txn->sender_pid);
            bio_put_ref(reply, handle);
            return 0;

        case SVC_MGR_ADD_SERVICE:
            s = bio_get_string16(msg, &len); //服务名
            handle = bio_get_ref(msg); //服务实体在servicemanager中的handle
            allow_isolated = bio_get_uint32(msg) ? 1 : 0;
             //注册指定服务
            if (do_add_service(bs, s, len, handle, txn->sender_euid,
                allow_isolated, txn->sender_pid))
                return -1;
            break;

        case SVC_MGR_LIST_SERVICES: {  
            uint32_t n = bio_get_uint32(msg);
            if (!svc_can_list(txn->sender_pid)) {
                return -1;
            }
            si = svclist;
            while ((n-- > 0) && si)
                si = si->next;
            if (si) {
                bio_put_string16(reply, si->name);
                return 0;
            }
            return -1;
        }
        }

        bio_put_uint32(reply, 0);
        return 0;
    }

该方法的功能：查询服务，注册服务，以及列举所有服务。不同的code对应不同的工作，定义在IBinder.h文件，跟IServiceManager.h中定义的code具有一一对应关系，具体关系如下所示。

|code|IBinder.h|IServiceManager.h|
|---|---|---|
|1|SVC_MGR_GET_SERVICE|GET_SERVICE_TRANSACTION|
|2|SVC_MGR_CHECK_SERVICE|CHECK_SERVICE_TRANSACTION|
|3|SVC_MGR_ADD_SERVICE|ADD_SERVICE_TRANSACTION|
|4|SVC_MGR_LIST_SERVICES|LIST_SERVICES_TRANSACTION|

servicemanager进程里面有一个链表svclist，记录着所有注册的服务svcinfo，每一个服务用svcinfo结构体来表示，该handle值是在注册服务的过程中，由服务所在进程那一端所确定的。svcinfo结构体代码如下所示。

    struct svcinfo
    {
        struct svcinfo *next;
        uint32_t handle; //服务的handle值
        struct binder_death death;
        int allow_isolated;
        size_t len; //服务名的长度
        uint16_t name[0]; //服务名
    };


如图5-3所示，整个ServiceManager启动过程的完整流程，这里整个过程都的都离不开Binder驱动层的实现。

![ServiceManager启动过程](/images/book/binder/5-2-2.start_service_manager.jpg)



### 5.2.2 获取ServiceManager代理

不论是注册服务，还是查询服务，都需要先向ServiceManager服务发起binder请求，获取ServiceManager服务的代理，
下面先来讲讲如何获取ServiceManager服务代理。ServiceManager代理可通过defaultServiceManager()方法，得到gDefaultServiceManager对象。
对于gDefaultServiceManager对象，如果存在则直接返回；如果不存在则创建该对象，创建过程包括调用open()打开binder驱动设备，利用mmap()映射内核的地址空间。

    // IServiceManager.cpp
    sp<IServiceManager> defaultServiceManager()
    {
        if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
        {
            AutoMutex _l(gDefaultServiceManagerLock); //加锁
            while (gDefaultServiceManager == NULL) {
                gDefaultServiceManager = interface_cast<IServiceManager>(
                    ProcessState::self()->getContextObject(NULL));
                if (gDefaultServiceManager == NULL)
                    sleep(1);
            }
        }
        return gDefaultServiceManager;
    }

获取ServiceManager对象采用**单例模式**，当gDefaultServiceManager存在，则直接返回，否则创建一个新对象。 此处与一般的单例模式不太一样，里面多了一层while循环，这是google在2013年1月Todd Poynor提交的修改。当尝试创建或获取ServiceManager时，ServiceManager可能尚未准备就绪，这时通过sleep 1秒后，循环尝试获取直到成功。

gDefaultServiceManager的创建过程，如图5-4所示，可分解为3个步骤：

- ProcessState::self()：用于创建ProcessState对象，每个进程有且只有一个ProcessState对象，存在则直接返回，不存在则创建;
- getContextObject()： 用于创建BpBinder对象，对于handle=0的BpBinder对象，存在则直接返回，不存在才创建;
- interface_cast<IServiceManager>()：用于创建BpServiceManager对象;

![get_servicemanager](/images/binder/get_servicemanager/get_servicemanager.jpg)


#### 1. 创建ProcessState对象

    // ProcessState.cpp
    sp<ProcessState> ProcessState::self()
    {
        Mutex::Autolock _l(gProcessMutex);
        if (gProcess != NULL) {
            return gProcess;
        }
        //实例化ProcessState
        gProcess = new ProcessState("/dev/binder");
        return gProcess;
    }

采用单例模式获得ProcessState对象，从而保证每一个进程只有一个ProcessState对象。其中`gProcess`和`gProcessMutex`是保存在`Static.cpp`类的全局变量。

    ProcessState::ProcessState(const char *driver)
        : mDriverName(String8(driver))
        , mDriverFD(open_driver(driver)) // 打开Binder驱动
        , mVMStart(MAP_FAILED)
        , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
        , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
        , mExecutingThreadsCount(0)
        , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
        , mManagesContexts(false)
        , mBinderContextCheckFunc(NULL)
        , mBinderContextUserData(NULL)
        , mThreadPoolStarted(false)
        , mThreadPoolSeq(1)
    {
        if (mDriverFD >= 0) {
            //采用内存映射函数mmap，给binder分配一块虚拟地址空间,用来接收事务
            mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
            if (mVMStart == MAP_FAILED) {
                close(mDriverFD); //没有足够空间分配给/dev/binder,则关闭驱动
                mDriverFD = -1;
            }
        }
    }

ProcessState的单例模式，保证一个进程只能打开binder设备一次,其中ProcessState的成员变量`mDriverFD`记录binder驱动的fd，用于访问binder设备。ProcessState初始化主要是跟Binder驱动打交道，交互内容：

- open()，打开Binder驱动，将mDriverFD记录binder驱动的fd，用于访问binder设备；
- ioctl()，参数为BINDER_SET_MAX_THREADS，设置binder默认的最大可并发访问的线程数为15+1=16个；
- mmap(), 分配用于处理事务的一块虚拟内存，设置大小为(1*1024*1024) - (4096 *2)=1016KB；

再来具体看看open_driver过程

    // ProcessState.cpp
    static int open_driver(const char *driver)
    {
        // 打开binder驱动设备，建立与内核的Binder驱动的交互通道
        int fd = open(driver, O_RDWR);
        if (fd >= 0) {
            fcntl(fd, F_SETFD, FD_CLOEXEC);
            int vers = 0;
            status_t result = ioctl(fd, BINDER_VERSION, &vers);
            if (result == -1) {
                close(fd);
                fd = -1;
            }
            if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
                close(fd);
                fd = -1;
            }
            size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
            // 通过ioctl设置binder驱动，能支持的最大线程数
            result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        }
        return fd;
    }

open_driver过程，主要是打开binder驱动设备，验证binder版本是否一致，设置binder支持的最大线程数。

#### 2. 创建BpBinder对象

通过ProcessState的getContextObject()方法来获取BpBinder对象，代码如下：

    // ProcessState.cpp
    sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
    {
        return getStrongProxyForHandle(0);
    }

获取handle=0的IBinder，再来看看getStrongProxyForHandle()过程，代码如下：

    sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
    {
        sp<IBinder> result;
        AutoMutex _l(mLock);
        //查找handle对应的资源项
        handle_entry* e = lookupHandleLocked(handle);

        if (e != NULL) {
            IBinder* b = e->binder;
            if (b == NULL || !e->refs->attemptIncWeak(this)) {
                if (handle == 0) {
                    Parcel data;
                    //通过ping操作测试binder是否准备就绪
                    status_t status = IPCThreadState::self()->transact(
                            0, IBinder::PING_TRANSACTION, data, NULL, 0);
                    if (status == DEAD_OBJECT)
                       return NULL;
                }
                //当handle值所对应的IBinder不存在或弱引用无效时，则创建BpBinder对象
                b = new BpBinder(handle);
                e->binder = b;
                if (b) e->refs = b->getWeakRefs();
                result = b;
            } else {
                result.force_set(b); //当找到目标handle所对应的BpBinder，则直接返回
                e->refs->decWeak(this);
            }
        }
        return result;
    }

每个ProcessState里面都有一个mHandleToObject向量，记录着handle以及相对应的handle_entry结构体，向量索引号跟handle值相等， 如图5-5所示。

![mHandleToObject向量](/images/book/binder/5-2-3-mHandleToObject.jpg)



当目标handle值在mHandleToObject中找不到相应的BpBinder，则会创建新的BpBinder对象。BpBinder通过handle来指向所对应BBinder, 在整个Binder系统中handle=0代表ServiceManager所对应的BBinder。
针对handle==0的特殊情况，通过PING_TRANSACTION来判断servicemanager是否准备就绪，用于确保servicemanger进程先启动再执行后面的操作。对于查询handle过程，见代码如下：

    ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
    {
        const size_t N=mHandleToObject.size();
        //当handle大于mHandleToObject的长度时，进入该分支
        if (N <= (size_t)handle) {
            handle_entry e;
            e.binder = NULL;
            e.refs = NULL;
            //从mHandleToObject的第N个位置开始，插入(handle+1-N)个e到队列中
            status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
            if (err < NO_ERROR) return NULL;
        }
        return &mHandleToObject.editItemAt(handle);
    }

根据handle值来查找对应的`handle_entry`,`handle_entry`是一个结构体，里面记录IBinder和weakref_type两个指针。当handle大于mHandleToObject的Vector长度时，则向该Vector中添加(handle+1-N)个handle_entry结构体，然后再返回handle向对应位置的handle_entry结构体指针。


再来看看BpBinder的创建过程，代码如下：

    // BpBinder.cpp
    BpBinder::BpBinder(int32_t handle)
        : mHandle(handle)
        , mAlive(1)
        , mObitsSent(0)
        , mObituaries(NULL)
    {
        extendObjectLifetime(OBJECT_LIFETIME_WEAK); //延长对象的生命时间
        IPCThreadState::self()->incWeakHandle(handle); //handle所对应的binder弱引用 + 1
    }

设置BpBinder的生命周期类型为OBJECT_LIFETIME_WEAK，该类型特点是当强引用计数为0，弱应用计数不为0时，实际对象并不会被回收；
只有当弱引用计数也减到0时，实际对象和weakref_impl对象会同时被回收。

incWeakHandle(handle)过程，向Binder驱动写入对BC_INCREFS和handle信息，其该handle所对应的Binder的弱引用增加1.

#### 3. 创建BpServiceManager对象

    // IInterface.h
    template<typename INTERFACE>
    inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
    {
        return INTERFACE::asInterface(obj);
    }

这是一个模板函数，可得出interface_cast<IServiceManager>() 等价于 IServiceManager::asInterface(), 接下来的难点是asInterface()函数，通过搜索代码会发现根本找不到这个方法是在哪里定义的。其实是通过模板函数来定义的，通过下面两个代码完成的：

    //IServiceManager.h
    DECLARE_META_INTERFACE(ServiceManager)
    //IServiceManager.cpp
    IMPLEMENT_META_INTERFACE(ServiceManager,"android.os.IServiceManager")

先来看看DECLARE_META_INTERFACE的定义，代码如下：

    //IInterface.h
    #define DECLARE_META_INTERFACE(INTERFACE)                               \
        static const android::String16 descriptor;                          \
        static android::sp<I##INTERFACE> asInterface(                       \
                const android::sp<android::IBinder>& obj);                  \
        virtual const android::String16& getInterfaceDescriptor() const;    \
        I##INTERFACE();                                                     \
        virtual ~I##INTERFACE();                                            \

将INTERFACE=ServiceManager，代入上述模块，展开即可得：

    static const android::String16 descriptor;
    static android::sp< IServiceManager > asInterface(const android::sp<android::IBinder>& obj)
    virtual const android::String16& getInterfaceDescriptor() const;
    IServiceManager ();
    virtual ~IServiceManager();

可见，该过程主要是声明IServiceManager的构造和析构方法，以及asInterface()和getInterfaceDescriptor()方法.

同理，IInterface.h定义了IMPLEMENT_META_INTERFACE模块，
将INTERFACE=ServiceManager, NAME="android.os.IServiceManager"代入后展开即可得：

    const android::String16 IServiceManager::descriptor(“android.os.IServiceManager”);

    const android::String16& IServiceManager::getInterfaceDescriptor() const
    {
         return IServiceManager::descriptor;
    }

     android::sp<IServiceManager> IServiceManager::asInterface(const android::sp<android::IBinder>& obj)
    {
           android::sp<IServiceManager> intr;
            if(obj != NULL) {
               intr = static_cast<IServiceManager *>(
                   obj->queryLocalInterface(IServiceManager::descriptor).get());
               if (intr == NULL) {
                   intr = new BpServiceManager(obj); //创建BpServiceManager对象
                }
            }
           return intr;
    }

IServiceManager::asInterface(obj), 此处参数obj为BpBinder，该对象的queryLocalInterface()返回值为NULL，所以等价于
 new BpServiceManager(BpBinder)。

BpServiceManager初始化的过程，先初始化父类对象。

    // IServiceManager.cpp
    BpServiceManager(const sp<IBinder>& impl)
        : BpInterface<IServiceManager>(impl)
    {    }

    // IInterface.h
    inline BpRefBase<IServiceManager>::BpInterface(const sp<IBinder>& remote)
        :BpRefBase(remote)
    {    }

    // Binder.cpp
    BpRefBase::BpRefBase(const sp<IBinder>& o)
        : mRemote(o.get()), mRefs(NULL), mState(0)
    {
        extendObjectLifetime(OBJECT_LIFETIME_WEAK);
        if (mRemote) {
            mRemote->incStrong(this);
            mRefs = mRemote->createWeak(this);
        }
    }

BpServiceManager对象初始化过程中，依次BpRefBase，BpRefBase，BpServiceManager的构造函数，赋予BpRefBase的mRemote的值为BpBinder(0)。以及增加BpBinder的强引用计数和弱引用计数，并执行BpBinder的onFirstRef(), 代码如下：

    void BpBinder::onFirstRef()
    {
        IPCThreadState* ipc = IPCThreadState::self();
        if (ipc) ipc->incStrongHandle(mHandle);
    }

incStrongHandle(handle)过程，向Binder驱动写入对BC_ACQUIRE和handle信息，其该handle所对应的Binder的强引用增加1.


可见，defaultServiceManager 等价于 new BpServiceManager(new BpBinder(0));

ProcessState::self()主要工作：

- 调用open()，打开/dev/binder驱动设备
- 调用mmap()，创建大小为1016KB的内存地址空间
- 设定当前进程最大的最大并发Binder线程个数为16

BpServiceManager作为跟servicemanager进程通信的代理类，创建过程：

- 创建BpBinder对象，向mOut写入BC_INCREFS;
- 创建BpServiceManager对象，向mOut写入BC_ACQUIRE;
- BpServiceManager通过继承接口IServiceManager实现了接口中的业务逻辑函数；
通过成员变量`mRemote`= new BpBinder(0)进行Binder通信工作。



> Tips: Native层的Binder架构,通过如下两个宏, 非常方便地创建了`new Bp##INTERFACE(obj)`，以及申明asInterface(),getInterfaceDescriptor()
>
> #define DECLARE_META_INTERFACE(INTERFACE)
>
> #define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)


获取ServiceManager代理的全过程， 如图5-5所示。

![获取ServiceManager代理](/images/book/binder/5-2-4-get_service_manager.jpg)

- open: 创建binder_proc
- BINDER_SET_MAX_THREADS: 设置proc->max_threads
- mmap: 创建创建binder_buffer


### 5.2.3 Binder框架核心类

#### 1. ProcessState类

到此这，初步接触到ProcessState，每个进程都有唯一的一个ProcessState对象，该对象很重要，先来看看该对象的成员变量和方法，类图所示

![ProcessState类图](/images/book/binder/5-2-5-ProcessState.jpg)



常用成员方法：

- self()：用于初始化mDriverName等于”/dev/binder“的ProcessState对象；
- initWithDriver(): 这是Android 8.0新加的方法，binder的不同/dev节点对应不同的binder域, 则进程需要通过选择binder设备节点进行交互，并将设备节点信息保存到mDriverName;
- getContextObject：用于获取handle=0的BpBinder的强指针；
- getStrongProxyForHandle()：用于获取指定handle所对应Bpbinder对象的强指针；
- getWeakProxyForHandle()：用于获取指定handle所对应Bpbinder对象的弱指针；
- startThreadPool(): 启动Binder线程池，设置mThreadPoolStarted=true，再启动binder主线程，唯一由用户主动创建的binder线程，且不可退出；
- spawnPooledThread(): 创建binder线程，然后执行IPCThreadState的joinThreadPool()方法，进入等待处理binder事务的状态；
当收到Binder驱动发送过来的BR_SPAWN_LOOPER协议，则会触发创建binder线程；当binder主线程，只能通过startThreadPool()方法来创建，并且只能创建一次。
- setThreadPoolMaThreadCount()：设置binder线程池的线程个数上限。 bindr线程名都有共同的规范为"Binder:进程号_序号"，序号从1开始，采用十六进制，比如pid=1000的创建的第10个binder线程名为”Binder:1000_A“。

常用成员变量：

- mDriverName：Binder设备节点名，比如/dev/binder, /dev/hwbinder, /dev/vndbinder；
- mDriverFD：执行open()操作后，打开的Binder设备驱动的文件描述符；
- mVMStart：执行mmap()操作后，已映射内存地址的指针；
- mExecutingThreadsCount：正处于执行IPCThreadState的executeCommand()方法的binder线程个数；每当binder线程执行完则会执行减1操作；
- mMaxThreads：设置binder线程池的线程个数上限，默认值为15，需要注意的是此处15并不包括Binder主线程的，所以真正的binder线程个数是16个。另外可通过setThreadPoolMaThreadCount设置binder线程池的个数
- mStarvationStartTimeMs：当Binder线程池的线程都处于工作状态，没有空闲的binder线程的时间点，称为binder饥饿开始时刻
- mHandleToObject：以向量的数据结构记录着一系列的handle_entry结构体，每个handle_entry记录着BpBinder对象和相应的weakref_impl对象；


在ProcessState过程不断地提及IPCThreadState类，接下来需要解开其面纱。

#### 2. IPCThreadState类

![IPCThreadState类](/images/book/binder/5-2-6-IPCThreadState.jpg)


常用成员方法：

- self()：初始化IPCThreadState对象，设置mIn和mOut的大小上限为256
- getCallingPid()/getCallingUid(): 获取远程发起Binder调用进程的pid/uid;
- clearCallingIdentity()/restoreCallingIdentity(): 清除/恢复远程调用者的pid和uid；
- joinThreadPool(): 从名字来看加入线程池，这里的线程池是指Binder线程池，加入该线程池后就只能处理binder事务。一类线程是通过spawnPooledThread()方法创建，线程名以”Binder:xxx"，线程创建后会主动调用joinThreadPool()；另一类是用户自行创建线程后，名字可自定义，然后再调用joinThreadPool()方法。工作内容一致，线程名不同，且第二类不受binder线程池上限的控制，也就是说binder线程个数是可以超过上限的。
- transact()：客户端发起Binder请求，
- writeTransactionData()：将Binder通信数据包Parcel封装到binder_transaction_data结构体，写入mOut
- talkWithDriver()：将mOut,mIn封装到binder_write_read结构体，通过ioctl向binder驱动发送BINDER_WRITE_READ命令
- waitForResponse()：将数据写入Binder驱动，并等待对端的响应，当收到满足条件的BR_XX协议则退出等待过程
- executeCommand()：处理BR_XX协议
- freeBuffer()：释放内存，当执行完BR_TRANSACTION或BR_REPLY的过程都会需要执行该操作
- requestDeathNotification()：注册死亡通知，将BC_REQUEST_DEATH_NOTIFICATION，写入mOut
- clearDeathNotification()：注销死亡通知，将BC_CLEAR_DEATH_NOTIFICATION，写入mOut


waitForResponse() 处于循环等待过程，直到收到以下任一BR，则会退出等待:

- BR_DEAD_REPLY
- BR_FAILED_REPLY
- BR_REPLY：Binder驱动向Client端发送回复数据，参数binder_transaction_data
- BR_TRANSACTION_COMPLETE：仅当发生异步binder调用，则直接退出等待

另外，还有在执行剩余其他BR协议过程发生异常也直接退出等待，再来看看executeCommand()处理的常见BR协议：

- BR_TRANSACTION：Binder驱动向Server端发送请求数据，参数binder_transaction_data
- BR_DEAD_BINDER：Binder驱动向Client端发送死亡通知
- BR_CLEAR_DEATH_NOTIFICATION_DONE：注销死亡通知完成
- BR_SPAWN_LOOPER：创建新的Looper线程
- BR_ERROR：发生错误
- BR_INCREFS，BR_DECREFS，BR_ACQUIRE，BR_RELEASE： 用于强弱引用计数的增减操作


#### 3. Parcel类

Parcel用于封装Binder通信过程的数据，类图所示

![Parcel类图](/images/book/binder/5-2-7-Parcel.jpg)


（1）成员变量：

- mData: 数据的起始地址， 所对应的方法是ipcData()
- mDataSize: 当前已存储的数据大小, 所对应的方法是ipcDataSize()
- mDataPos: 当前已读数据所在的位置， 当mDataPos<mDataSize，则说明还有数据可读
- mDataCapacity: 当前Parcel可容纳的数据大小上限，当容量不够时每次扩容约50%
- mObjects: flat_binder_object结构体的偏移量(相对mData的偏移量)， 所对应的方法是ipcObjects()
- mObjectsSize: 当前flat_binder_object结构体的个数， 所对应的方法是ipcObjectsCount()
- mObjectsCapacity: 当前Parcel可容纳的对象个数上限，当容量不够时每次扩容约50%
- mOwner: 代表释放函数relFunc， 可通过ipcSetDataReference()方法赋值
- mOwnerCookie：释放函数所需信息

（2） 扁平化过程：

- flatten_binder()： 将IBinder对象转换成扁平化对象flat_binder_object
    - 对于本地服务对象，则类型为BINDER_TYPE_BINDER， cookie记录binder对象地址；
    - 对于远程服务代理，则类型为BINDER_TYPE_HANDLE， handle记录binder代理信息；
- unflatten_binder()： 将flat_binder_object转换成IBinder对象；


（3） writeStrongBinder

在Binder通信过程，往往需要把Binder服务对象传递到对端，这时就需要使用writeStrongBinder， 代码如下：

    //parcel.cpp
    status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
    {
        return flatten_binder(ProcessState::self(), val, this);
    }

    status_t flatten_binder(const sp<ProcessState>& /*proc*/,
        const sp<IBinder>& binder, Parcel* out)
    {
        flat_binder_object obj;

        obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
        if (binder != NULL) {
            IBinder *local = binder->localBinder(); //本地Binder不为空
            if (!local) {
                BpBinder *proxy = binder->remoteBinder();
                const int32_t handle = proxy ? proxy->handle() : 0;
                obj.type = BINDER_TYPE_HANDLE;
                obj.binder = 0;
                obj.handle = handle;
                obj.cookie = 0;
            } else { //进入该分支
                obj.type = BINDER_TYPE_BINDER;
                obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
                obj.cookie = reinterpret_cast<uintptr_t>(local);
            }
        } else {
            ...
        }
        //将flat_binder_object类型的obj写入out
        return finish_flatten_binder(binder, obj, out);
    }


IBinder转换成flat_binder_object对象，有两种情况：

- 对于Binder实体,即type=BINDER_TYPE_BINDER，则cookie记录Binder实体的指针，binder记录Binder实体的引用计数对象；
- 对于Binder代理，即type=BINDER_TYPE_HANDLE，则handle记录Binder代理的句柄；


另外，关于localBinder，代码见Binder.cpp。

    BBinder* BBinder::localBinder()
    {
        return this;
    }

    BBinder* IBinder::localBinder()
    {
        return NULL;
    }

（4） ipcSetDataReference

ipcSetDataReference用于设置Parcel对象， 见如下代码：

    void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,
        const binder_size_t* objects, size_t objectsCount, release_func relFunc, void* relCookie)
    {
        binder_size_t minOffset = 0;
        freeDataNoInit(); // 初始化前需要释放数据，见下文
        mError = NO_ERROR;
        mData = const_cast<uint8_t*>(data);
        mDataSize = mDataCapacity = dataSize;
        mDataPos = 0;
        mObjects = const_cast<binder_size_t*>(objects);
        mObjectsSize = mObjectsCapacity = objectsCount;
        mNextObjectHint = 0;
        mOwner = relFunc;
        mOwnerCookie = relCookie;
        ...
    }

ipcSetDataReference()方法使用场景主要在收到BR_TRANSACTION或BR_REPLY的过程，需要释放相应的buffer，
此时mOwner便是IPCThreadState对象中的freeBuffer()方法，mOwnerCookie记录的便是IPCThreadState对象。

    void Parcel::freeDataNoInit()
    {
        if (mOwner) {
            ...
        } else {
            releaseObjects();  //释放对象
            if (mData) {
                pthread_mutex_lock(&gParcelGlobalAllocSizeLock);
                //更新已分配的Parcel对象个数和内存大小
                if (mDataCapacity <= gParcelGlobalAllocSize) {
                  gParcelGlobalAllocSize = gParcelGlobalAllocSize - mDataCapacity;
                } else {
                  gParcelGlobalAllocSize = 0;
                }
                if (gParcelGlobalAllocCount > 0) {
                  gParcelGlobalAllocCount--;
                }
                pthread_mutex_unlock(&gParcelGlobalAllocSizeLock);
                free(mData); //释放mData
            }
            if (mObjects) free(mObjects); //释放mObjects
        }
    }

gParcelGlobalAllocSize记录当前进程已分配的Parcel所占的内存大小，gParcelGlobalAllocCount记录当前进程已分配的Parcel对象个数。这两个数据可通过dumpsys meminfo的过程输出，如图所示。
62个JavaBBinder对象，30个Binder代理对象，152个Parcel对象个数，152个Parcel所占内存大小为38KB, 4个JavaDeathRecipient对象

![create_servicemanager](/images/book/binder/5-2-8-dump_meminfo_for_binder.png)

再回到freeDataNoInit()过程，此时mOwner还没有赋值，则执行releaseObjects()过程， 见如下代码：

    void Parcel::releaseObjects()
    {
        const sp<ProcessState> proc(ProcessState::self());
        size_t i = mObjectsSize;
        uint8_t* const data = mData;
        binder_size_t* const objects = mObjects;
        while (i > 0) {
            i--;
            const flat_binder_object* flat = reinterpret_cast<
                    flat_binder_object*>(data+objects[i]);
            release_object(proc, *flat, this, &mOpenAshmemSize);
        }
    }

根据flat_binder_object的类型来决定减少相应的强弱引用， 见如下代码：

    static void release_object(const sp<ProcessState>& proc,
        const flat_binder_object& obj, const void* who, size_t* outAshmemSize)
    {
        switch (obj.type) {
            case BINDER_TYPE_BINDER:
                if (obj.binder) {
                    reinterpret_cast<IBinder*>(obj.cookie)->decStrong(who);
                }
                return;
            case BINDER_TYPE_WEAK_BINDER:
                if (obj.binder)
                    reinterpret_cast<RefBase::weakref_type*>(obj.binder)->decWeak(who);
                return;
            case BINDER_TYPE_HANDLE: {
                const sp<IBinder> b = proc->getStrongProxyForHandle(obj.handle);
                if (b != NULL) {
                    b->decStrong(who);
                }
                return;
            }
            case BINDER_TYPE_WEAK_HANDLE: {
                const wp<IBinder> b = proc->getWeakProxyForHandle(obj.handle);
                if (b != NULL) b.get_refs()->decWeak(who);
                return;
            }
            case BINDER_TYPE_FD: {
                ...
                return;
            }
        }
    }


前面给mOwner赋值为freeBuffer，那么什么时候调用呢？ 答案就是在Parcel对象析构的时候，

    Parcel::~Parcel()
    {
        freeDataNoInit();
    }

    void Parcel::freeDataNoInit()
    {
        if (mOwner) { //此处mOwner等于freeBuffer
            mOwner(this, mData, mDataSize, mObjects, mObjectsSize, mOwnerCookie);
        } else {
            ...
        }
    }


释放过程就是向Binder驱动写入BC_FREE_BUFFER命令， 见代码如下：

    void IPCThreadState::freeBuffer(Parcel* parcel, const uint8_t* data,
                                    size_t /*dataSize*/,
                                    const binder_size_t* /*objects*/,
                                    size_t /*objectsSize*/, void* /*cookie*/)
    {
        if (parcel != NULL) parcel->closeFileDescriptors();
        IPCThreadState* state = self();
        state->mOut.writeInt32(BC_FREE_BUFFER);
        state->mOut.writePointer((uintptr_t)data);
    }


ipcSetDataReference()过程主要的功能是创建一个跟发送端一样的Parcel对象，并设定该对象的回收方法。


另外再介绍一些常见的核心方法。

|方法|功能|所属文件|
|---|---|---|
|writeStrongBinder|将BpBinder或BBinder转换成flat_binder_object对象|parcel.cpp|
|readStrongBinder|将flat_binder_object对象转换为BpBinder或BBinder|parcel.cpp|
|getStrongProxyForHandle|获取handle所对应Bpbinder对象的强指针sp|ProcessState.cpp|
|getWeakProxyForHandle|获取handle所对应Bpbinder对象的弱指针wp|ProcessState.cpp|
|javaObjectForIBinder|将BpBinder对象转换为BinderProxy对象|android_util_binder.cpp|
|ibinderForJavaObject|将Java层Binder或BinderProxy转换为Native层的IBinder|android_util_binder.cpp|
|parcelForJavaObject|Parcel(Java)转换为Parcel(C++)|android_os_Parcel.cpp|
|localBinder|BpBinder则返回NULL; BBinder则返回this指针|Binder.cpp|
