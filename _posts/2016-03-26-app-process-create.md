---
layout: post
title:  "理解Android进程创建流程"
date:   2016-03-26 21:10:11
catalog:  true
tags:
    - android
    - 进程系列

---

> 基于Android 6.0的源码剖析， 分析Android进程是如何一步步创建的，本文涉及到的源码：

    /frameworks/base/core/java/com/android/internal/os/
        - ZygoteInit.java
        - ZygoteConnection.java
        - RuntimeInit.java
        - Zygote.java

    /frameworks/base/core/java/android/os/Process.java
    /frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
    /frameworks/base/core/jni/AndroidRuntime.cpp
    /frameworks/base/cmds/app_process/App_main.cpp （内含AppRuntime类）

    /bionic/libc/bionic/fork.cpp
    /bionic/libc/bionic/pthread_atfork.cpp
    
    /libcore/dalvik/src/main/java/dalvik/system/ZygoteHooks.java
    /art/runtime/native/dalvik_system_ZygoteHooks.cc
    /art/runtime/Runtime.cc
    /art/runtime/Thread.cc
    /art/runtime/signal_catcher.cc


## 一. 概述

### 准备知识

本文要介绍的是进程的创建，先简单说说进程与线程的区别。

**进程：**每个`App`在启动前必须先创建一个进程，该进程是由`Zygote` fork出来的，进程具有独立的资源空间，用于承载App上运行的各种Activity/Service等组件。进程对于上层应用来说是完全透明的，这也是google有意为之，让App程序都是运行在Android Runtime。大多数情况一个`App`就运行在一个进程中，除非在AndroidManifest.xml中配置`Android:process`属性，或通过native代码fork进程。

**线程：**线程对应用开发者来说非常熟悉，比如每次`new Thread().start()`都会创建一个新的线程，该线程并没有自己独立的地址空间，而是与其所在进程之间资源共享。从Linux角度来说进程与线程都是一个task_struct结构体，除了是否共享资源外，并没有其他本质的区别。

在接下来的文章，会涉及到system_server进程和Zygote进程，下面简要这两个进程：

- `system_server`进程：是用于管理整个Java framework层，包含ActivityManager，PowerManager等各种系统服务;
- `Zygote`进程：是Android系统的首个Java进程，Zygote是所有Java进程的父进程，包括 `system_server`进程以及所有的App进程都是Zygote的子进程，注意这里说的是子进程，而非子线程。

如果想更进一步了解system_server进程和Zygote进程在整个Android系统所处的地位，可查看我的另一个文章[Android系统-开篇](http://gityuan.com/android)。

### 进程创建图

对于大多数的应用开发者来说创建线程比较熟悉，而对于创建进程并没有太多的概念。对于系统工程师或者高级开发者，还是有很必要了解Android系统是如何一步步地创建出一个进程的。先来看一张进程创建过程的简要图：

![start_app_process](/images/android-process/start_app_process.jpg)

图解：

1. **App发起进程**：当从桌面启动应用，则发起进程便是Launcher所在进程；当从某App内启动远程进程，则发送进程便是该App所在进程。发起进程先通过binder发送消息给system_server进程；
2. **system_server进程**：调用Process.start()方法，通过socket向zygote进程发送创建新进程的请求；
3. **zygote进程**：在执行`ZygoteInit.main()`后便进入`runSelectLoop()`循环体内，当有客户端连接时便会执行ZygoteConnection.runOnce()方法，再经过层层调用后fork出新的应用进程；
4. **新进程**：执行handleChildProc方法，最后调用ActivityThread.main()方法。

接下来，依次从`system_server进程发起请求`到`Zygote创建进程`，再到`新进程的运行`这3大块展开讲解进程创建是一个怎样的过程。

## 二. system_server发起请求

### 1. Process.start

[-> Process.java]

    public static final ProcessStartResult start(final String processClass,
                              final String niceName,
                              int uid, int gid, int[] gids,
                              int debugFlags, int mountExternal,
                              int targetSdkVersion,
                              String seInfo,
                              String abi,
                              String instructionSet,
                              String appDataDir,
                              String[] zygoteArgs) {
        try {
             //【见小节2】
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            throw new RuntimeException("");
        }
    }

### 2. startViaZygote

[-> Process.java]

    private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        synchronized(Process.class) {
            ArrayList<String> argsForZygote = new ArrayList<String>();

            argsForZygote.add("--runtime-args");
            argsForZygote.add("--setuid=" + uid);
            argsForZygote.add("--setgid=" + gid);
            argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

            if (niceName != null) {
                argsForZygote.add("--nice-name=" + niceName);
            }
            if (appDataDir != null) {
                argsForZygote.add("--app-data-dir=" + appDataDir);
            }
            argsForZygote.add(processClass);

            if (extraArgs != null) {
                for (String arg : extraArgs) {
                    argsForZygote.add(arg);
                }
            }
             //【见小节3】
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }

该过程主要工作是生成`argsForZygote`数组，该数组保存了进程的uid、gid、groups、target-sdk、nice-name等一系列的参数。


### 3. zygoteSendArgsAndGetResult
[-> Process.java]

    private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            //其中zygoteState 【见小节3.1】
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            writer.write(Integer.toString(args.size()));
            writer.newLine();

            int sz = args.size();
            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                if (arg.indexOf('\n') >= 0) {
                    throw new ZygoteStartFailedEx(
                            "embedded newlines not allowed");
                }
                writer.write(arg);
                writer.newLine();
            }

            writer.flush();

            ProcessStartResult result = new ProcessStartResult();
            //等待socket服务端（即zygote）返回新创建的进程pid;
            //对于等待时长问题，Google正在考虑此处是否应该有一个timeout，但目前是没有的。
            result.pid = inputStream.readInt();
            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            result.usingWrapper = inputStream.readBoolean();
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }

这个方法的主要功能是通过socket通道向Zygote进程发送一个参数列表，然后进入阻塞等待状态，直到远端的socket服务端发送回来新创建的进程pid才返回。

#### 3.1 openZygoteSocketIfNeeded

    private static ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                //向主zygote发起connect()操作
                primaryZygoteState = ZygoteState.connect(ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                ...
            }
        }

        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            //当主zygote没能匹配成功，则采用第二个zygote，发起connect()操作
            secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
        }

        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }
        ...
    }

`openZygoteSocketIfNeeded(abi)`方法是根据当前的abi来选择与zygote还是zygote64来进行通信。


既然system_server进程的zygoteSendArgsAndGetResult()方法通过socket向Zygote进程发送消息，这是便会唤醒Zygote进程，来响应socket客户端的请求（即system_server端），接下来的操作便是在Zygote来创建进程【见小节4】

## 三. Zygote创建进程

文章[Android系统启动-zygote篇](http://gityuan.com/2016/02/13/android-zygote/)已介绍，简单来说就是Zygote进程是由由init进程而创建的，进程启动之后调用ZygoteInit.main()方法，经过创建socket管道，预加载资源后，便进程runSelectLoop()方法。

### 4. ZygoteInit.main

[-->ZygoteInit.java]

    public static void main(String argv[]) {
        try {
            runSelectLoop(abiList); //【见小节5】
            ....
        } catch (MethodAndArgsCaller caller) {
            caller.run(); //【见小节16】
        } catch (RuntimeException ex) {
            closeServerSocket();
            throw ex;
        }
    }

后续会讲到runSelectLoop()方法会抛出异常`MethodAndArgsCaller`，从而进入caller.run()方法。

### 5. runSelectLoop
[-> ZygoteInit.java]

    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
        //sServerSocket是socket通信中的服务端，即zygote进程。保存到fds[0]
        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                 //处理轮询状态，当pollFds有事件到来则往下执行，否则阻塞在这里
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                ...
            }

            for (int i = pollFds.length - 1; i >= 0; --i) {
                //采用I/O多路复用机制，当接收到客户端发出连接请求 或者数据处理请求到来，则往下执行；
                // 否则进入continue，跳出本次循环。
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                    //即fds[0]，代表的是sServerSocket，则意味着有客户端连接请求；
                    // 则创建ZygoteConnection对象,并添加到fds。//【见小节5.1】
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor()); //添加到fds.
                } else {
                    //i>0，则代表通过socket接收来自对端的数据，并执行相应操作【见小节6】
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i); //处理完则从fds中移除该文件描述符
                    }
                }
            }
        }
    }

该方法主要功能：

- 客户端通过openZygoteSocketIfNeeded()来跟zygote进程建立连接。zygote进程收到客户端连接请求后执行accept()；然后再创建ZygoteConnection对象,并添加到fds数组列表；
- 建立连接之后，可以跟客户端通信，进入runOnce()方法来接收客户端数据，并执行进程创建工作。

#### 5.1 acceptCommandPeer
[-> ZygoteInit.java]

    private static ZygoteConnection acceptCommandPeer(String abiList) {
        try {
            return new ZygoteConnection(sServerSocket.accept(), abiList);
        } catch (IOException ex) {
            ...
        }
    }

接收客户端发送过来的connect()操作，Zygote作为服务端执行accept()操作。
再后面客户端调用write()写数据，Zygote进程调用read()读数据。


没有连接请求时会进入休眠状态，当有创建新进程的连接请求时，唤醒Zygote进程，创建Socket通道ZygoteConnection，然后执行ZygoteConnection的runOnce()方法。

### 6. runOnce

[-> ZygoteConnection.java]

    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

        String args[];
        Arguments parsedArgs = null;
        FileDescriptor[] descriptors;

        try {
            //读取socket客户端发送过来的参数列表
            args = readArgumentList();
            descriptors = mSocket.getAncillaryFileDescriptors();
        } catch (IOException ex) {
            closeSocket();
            return true;
        }

        PrintStream newStderr = null;
        if (descriptors != null && descriptors.length >= 3) {
            newStderr = new PrintStream(new FileOutputStream(descriptors[2]));
        }

        int pid = -1;
        FileDescriptor childPipeFd = null;
        FileDescriptor serverPipeFd = null;

        try {
            //将binder客户端传递过来的参数，解析成Arguments对象格式
            parsedArgs = new Arguments(args);
            ...

            int [] fdsToClose = { -1, -1 };
            FileDescriptor fd = mSocket.getFileDescriptor();
            if (fd != null) {
                fdsToClose[0] = fd.getInt$();
            }

            fd = ZygoteInit.getServerSocketFileDescriptor();
            if (fd != null) {
                fdsToClose[1] = fd.getInt$();
            }
            fd = null;
            //【见小节7】
            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                    parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                    parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                    parsedArgs.appDataDir);
        } catch (Exception e) {
            ...
        }

        try {
            if (pid == 0) {
                //子进程执行
                IoUtils.closeQuietly(serverPipeFd);
                serverPipeFd = null;
                //【见小节13】
                handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);

                // 不应到达此处，子进程预期的是抛出异常ZygoteInit.MethodAndArgsCaller或者执行exec().
                return true;
            } else {
                //父进程执行
                IoUtils.closeQuietly(childPipeFd);
                childPipeFd = null;
                return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
            }
        } finally {
            IoUtils.closeQuietly(childPipeFd);
            IoUtils.closeQuietly(serverPipeFd);
        }
    }

### 7. forkAndSpecialize

[-> Zygote.java]

    public static int forkAndSpecialize(int uid, int gid, int[] gids, int debugFlags,
          int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
          String instructionSet, String appDataDir) {
        VM_HOOKS.preFork(); //【见小节8】
        int pid = nativeForkAndSpecialize(
                  uid, gid, gids, debugFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
                  instructionSet, appDataDir); //【见小节9】
        ...
        VM_HOOKS.postForkCommon(); //【见小节11】
        return pid;
    }

VM_HOOKS是Zygote对象的静态成员变量：VM_HOOKS = new ZygoteHooks();

#### 7.1 Zygote进程

先说说Zygote进程，如下图：
![zygote_sub_thread](/images/android-process/zygote_sub_thread.png)

从图中可知Zygote进程有4个Daemon子线程分别是ReferenceQueueDaemon，FinalizerDaemon，FinalizerWatchdogDaemon，HeapTaskDaemon。图中线程名显示的并不完整是由于底层的进程结构体`task_struct`是由长度为16的char型数组保存，超过15个字符便会截断。

可能有人会问zygote64进程不是还有system_server，com.android.phone等子线程，怎么会只有4个呢？那是因为这些并不是Zygote子线程，而是Zygote的子进程。在图中用红色圈起来的是进程的[VSIZE，virtual size)](http://gityuan.com/2015/10/11/ps-command/)，代表的是进程虚拟地址空间大小。线程与进程的最为本质的区别便是是否共享内存空间，图中VSIZE和Zygote进程相同的才是Zygote的子线程，否则就是Zygote的子进程。

### 8. preFork

[-> ZygoteHooks.java]

     public void preFork() {
        Daemons.stop(); //停止4个Daemon子线程【见小节8.1】
        waitUntilAllThreadsStopped(); //等待所有子线程结束【见小节8.2】
        token = nativePreFork(); //完成gc堆的初始化工作【见小节8.3】
    }

#### 8.1 Daemons.stop

    public static void stop() {
        HeapTaskDaemon.INSTANCE.stop(); //Java堆整理线程
        ReferenceQueueDaemon.INSTANCE.stop(); //引用队列线程
        FinalizerDaemon.INSTANCE.stop(); //析构线程
        FinalizerWatchdogDaemon.INSTANCE.stop(); //析构监控线程
    }

此处守护线程Stop方式是先调用目标线程interrrupt()方法，然后再调用目标线程join()方法，等待线程执行完成。

#### 8.2 waitUntilAllThreadsStopped

    private static void waitUntilAllThreadsStopped() {
        File tasks = new File("/proc/self/task");
        // 当/proc中线程数大于1，就出让CPU直到只有一个线程，才退出循环
        while (tasks.list().length > 1) {
            Thread.yield();
        }
    }

#### 8.3 nativePreFork
nativePreFork通过JNI最终调用如下方法：

[-> dalvik_system_ZygoteHooks.cc]

    static jlong ZygoteHooks_nativePreFork(JNIEnv* env, jclass) {
        Runtime* runtime = Runtime::Current();
        runtime->PreZygoteFork(); // 见下文
        if (Trace::GetMethodTracingMode() != TracingMode::kTracingInactive) {
          Trace::Pause();
        }
        //将线程转换为long型并保存到token，该过程是非安全的
        return reinterpret_cast<jlong>(ThreadForEnv(env));
    }

至于runtime->PreZygoteFork的过程：

    void Runtime::PreZygoteFork() {
        // 堆的初始化工作。这里就不继续再往下追art虚拟机
        heap_->PreZygoteFork();
    }

VM_HOOKS.preFork()的主要功能便是停止Zygote的4个Daemon子线程的运行，等待并确保Zygote是单线程（用于提升fork效率），并等待这些线程的停止，初始化gc堆的工作, 并将线程转换为long型并保存到token

### 9. nativeForkAndSpecialize
nativeForkAndSpecialize()通过JNI最终调用调用如下方法：

[-> com_android_internal_os_Zygote.cpp]

    static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
        JNIEnv* env, jclass, jint uid, jint gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits,
        jint mount_external, jstring se_info, jstring se_name,
        jintArray fdsToClose, jstring instructionSet, jstring appDataDir) {
        // 将CAP_WAKE_ALARM赋予蓝牙进程
        jlong capabilities = 0;
        if (uid == AID_BLUETOOTH) {
            capabilities |= (1LL << CAP_WAKE_ALARM);
        }
        //【见流程10】
        return ForkAndSpecializeCommon(env, uid, gid, gids, debug_flags,
                rlimits, capabilities, capabilities, mount_external, se_info,
                se_name, false, fdsToClose, instructionSet, appDataDir);
    }

### 10. ForkAndSpecializeCommon

[-> com_android_internal_os_Zygote.cpp]

    static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                         jint debug_flags, jobjectArray javaRlimits,
                                         jlong permittedCapabilities, jlong effectiveCapabilities,
                                         jint mount_external,
                                         jstring java_se_info, jstring java_se_name,
                                         bool is_system_server, jintArray fdsToClose,
                                         jstring instructionSet, jstring dataDir) {
      //设置子进程的signal信号处理函数
      SetSigChldHandler();
      //fork子进程 【见流程10.1】
      pid_t pid = fork();
      if (pid == 0) { //进入子进程
        DetachDescriptors(env, fdsToClose); //关闭并清除文件描述符

        if (!is_system_server) {
            //对于非system_server子进程，则创建进程组
            int rc = createProcessGroup(uid, getpid());
        }
        SetGids(env, javaGids); //设置设置group
        SetRLimits(env, javaRlimits); //设置资源limit

        int rc = setresgid(gid, gid, gid);
        rc = setresuid(uid, uid, uid);

        SetCapabilities(env, permittedCapabilities, effectiveCapabilities);
        SetSchedulerPolicy(env); //设置调度策略

         //selinux上下文
        rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);

        if (se_info_c_str == NULL && is_system_server) {
          se_name_c_str = "system_server";
        }
        if (se_info_c_str != NULL) {
          SetThreadName(se_name_c_str); //设置线程名为system_server，方便调试
        }
        //在Zygote子进程中，设置信号SIGCHLD的处理器恢复为默认行为
        UnsetSigChldHandler();
        //等价于调用zygote.callPostForkChildHooks() 【见流程10.2】
        env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, debug_flags,
                                  is_system_server ? NULL : instructionSet);
        ...

      } else if (pid > 0) {
        //进入父进程，即Zygote进程
      }
      return pid;
    }

#### 10.1 fork()

fork()采用copy on write技术，这是linux创建进程的标准方法，调用一次，返回两次，返回值有3种类型。

- 父进程中，fork返回新创建的子进程的pid;
- 子进程中，fork返回0；
- 当出现错误时，fork返回负数。（当进程数超过上限或者系统内存不足时会出错）

fork()的主要工作是寻找空闲的进程号pid，然后从父进程拷贝进程信息，例如数据段和代码段，fork()后子进程要执行的代码等。
Zygote进程是所有Android进程的母体，包括system_server和各个App进程。zygote利用fork()方法生成新进程，对于新进程A复用Zygote进程本身的资源，再加上新进程A相关的资源，构成新的应用进程A。其中下图中Zygote进程的libc、vm、preloaded classes、preloaded resources是如何生成的，可查看另一个文章[Android系统启动-zygote篇](http://gityuan.com/2016/02/13/android-zygote/#preload)，见下图：

![zygote_fork](/images/boot/zygote/zygote_fork.jpg)

copy-on-write过程：当父子进程任一方修改内存数据时（这是on-write时机），才发生缺页中断，从而分配新的物理内存（这是copy操作）。

copy-on-write原理：写时拷贝是指子进程与父进程的页表都所指向同一个块物理内存，fork过程只拷贝父进程的页表，并标记这些页表是只读的。父子进程共用同一份物理内存，如果父子进程任一方想要修改这块物理内存，那么会触发缺页异常(page fault)，Linux收到该中断便会创建新的物理内存，并将两个物理内存标记设置为可写状态，从而父子进程都有各自独立的物理内存。

##### 10.1.1 fork.cpp
[-> bionic/fork.cpp]

    #define FORK_FLAGS (CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID | SIGCHLD)
    int fork() {
      __bionic_atfork_run_prepare(); //[见小节2.1.1]

      pthread_internal_t* self = __get_thread();

      //fork期间，获取父进程pid，并使其缓存值无效
      pid_t parent_pid = self->invalidate_cached_pid();
      //系统调用【见小节2.2】
      int result = syscall(__NR_clone, FORK_FLAGS, NULL, NULL, NULL, &(self->tid));
      if (result == 0) {
        self->set_cached_pid(gettid());
        __bionic_atfork_run_child(); //fork完成执行子进程回调方法[见小节2.1.1]
      } else {
        self->set_cached_pid(parent_pid);
        __bionic_atfork_run_parent(); //fork完成执行父进程回调方法
      }
      return result;
    }

功能说明：在执行syscall的前后都有相应的回调方法。

- __bionic_atfork_run_prepare： fork完成前，父进程回调方法
- __bionic_atfork_run_child： fork完成后，子进程回调方法
- __bionic_atfork_run_paren： fork完成后，父进程回调方法

以上3个方法的实现都位于bionic/pthread_atfork.cpp。如果有需要，可以扩展该回调方法，添加相关的业务需求。


#### 10.2 Zygote.callPostForkChildHooks
[-> Zygote.java]

    private static void callPostForkChildHooks(int debugFlags, boolean isSystemServer,
            String instructionSet) {
        //调用ZygoteHooks.postForkChild()
        VM_HOOKS.postForkChild(debugFlags, isSystemServer, instructionSet);
    }

[-> ZygoteHooks.java]

    public void postForkChild(int debugFlags, String instructionSet) {
        //【见流程10.3】
        nativePostForkChild(token, debugFlags, instructionSet);
        Math.setRandomSeedInternal(System.currentTimeMillis());
    }

在这里，设置了新进程Random随机数种子为当前系统时间，也就是在进程创建的那一刻就决定了未来随机数的情况，也就是伪随机。

#### 10.3 nativePostForkChild
nativePostForkChild通过JNI最终调用调用如下方法：

[-> dalvik_system_ZygoteHooks.cc]

    static void ZygoteHooks_nativePostForkChild(JNIEnv* env, jclass, jlong token, jint debug_flags,
                                                jstring instruction_set) {
        //此处token是由[小节8.3]创建的，记录着当前线程
        Thread* thread = reinterpret_cast<Thread*>(token);
        //设置新进程的主线程id
        thread->InitAfterFork();
        ..
        if (instruction_set != nullptr) {
          ScopedUtfChars isa_string(env, instruction_set);
          InstructionSet isa = GetInstructionSetFromString(isa_string.c_str());
          Runtime::NativeBridgeAction action = Runtime::NativeBridgeAction::kUnload;
          if (isa != kNone && isa != kRuntimeISA) {
            action = Runtime::NativeBridgeAction::kInitialize;
          }
          //【见流程10.4】
          Runtime::Current()->DidForkFromZygote(env, action, isa_string.c_str());
        } else {
          Runtime::Current()->DidForkFromZygote(env, Runtime::NativeBridgeAction::kUnload, nullptr);
        }
    }

#### 10.4 DidForkFromZygote

[-> Runtime.cc]

    void Runtime::DidForkFromZygote(JNIEnv* env, NativeBridgeAction action, const char* isa) {
      is_zygote_ = false;
      if (is_native_bridge_loaded_) {
        switch (action) {
          case NativeBridgeAction::kUnload:
            UnloadNativeBridge(); //卸载用于跨平台的桥连库
            is_native_bridge_loaded_ = false;
            break;
          case NativeBridgeAction::kInitialize:
            InitializeNativeBridge(env, isa);//初始化用于跨平台的桥连库
            break;
        }
      }
      //创建Java堆处理的线程池
      heap_->CreateThreadPool();
      //重置gc性能数据，以保证进程在创建之前的GCs不会计算到当前app上。
      heap_->ResetGcPerformanceInfo();
      if (jit_.get() == nullptr && jit_options_->UseJIT()) {
        //当flag被设置，并且还没有创建JIT时，则创建JIT
        CreateJit();
      }
      //设置信号处理函数
      StartSignalCatcher();
      //启动JDWP线程，当命令debuger的flags指定"suspend=y"时，则暂停runtime
      Dbg::StartJdwp();
    }

关于信号处理过程，其代码位于signal_catcher.cc文件中，后续会单独讲解。

### 11. postForkCommon

[-> ZygoteHooks.java]

    public void postForkCommon() {
        Daemons.start();
    }

    public static void start() {
        ReferenceQueueDaemon.INSTANCE.start();
        FinalizerDaemon.INSTANCE.start();
        FinalizerWatchdogDaemon.INSTANCE.start();
        HeapTaskDaemon.INSTANCE.start();
    }

VM_HOOKS.postForkCommon的主要功能是在fork新进程后，启动Zygote的4个Daemon线程，java堆整理，引用队列，以及析构线程。


### 12. forkAndSpecialize小结

该方法主要功能：

- preFork： 停止Zygote的4个Daemon子线程的运行，初始化gc堆；
- nativeForkAndSpecialize：调用`fork()`创建新进程，设置新进程的主线程id，重置gc性能数据，设置信号处理函数等功能。
- postForkCommon：启动4个Deamon子线程。


其调用关系链：

    Zygote.forkAndSpecialize
        ZygoteHooks.preFork
            Daemons.stop
            ZygoteHooks.nativePreFork
                dalvik_system_ZygoteHooks.ZygoteHooks_nativePreFork
                    Runtime::PreZygoteFork
                        heap_->PreZygoteFork()
        Zygote.nativeForkAndSpecialize
            com_android_internal_os_Zygote.ForkAndSpecializeCommon
                fork()
                Zygote.callPostForkChildHooks
                    ZygoteHooks.postForkChild
                        dalvik_system_ZygoteHooks.nativePostForkChild
                            Runtime::DidForkFromZygote
        ZygoteHooks.postForkCommon
            Daemons.start

**时序图：** 点击查看[大图](http://gityuan.com/images/android-process/fork_and_specialize.jpg)

![fork_and_specialize](/images/android-process/fork_and_specialize.jpg)


到此App进程已完成了创建的所有工作，接下来开始新创建的App进程的工作。在前面ZygoteConnection.runOnce方法中，zygote进程执行完`forkAndSpecialize()`后，新创建的App进程便进入`handleChildProc()`方法，下面的操作运行在App进程。

## 四. 新进程运行

在前面[流程6]runOnce()过程中调用forkAndSpecialize()创建完新进程后，返回值pid=0(即运行在子进程)继续开始执行handleChildProc()方法。

### 13. handleChildProc

[-> ZygoteConnection.java]

    private void handleChildProc(Arguments parsedArgs,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
            throws ZygoteInit.MethodAndArgsCaller {

        //关闭Zygote的socket两端的连接
        closeSocket();
        ZygoteInit.closeServerSocket();

        if (descriptors != null) {
            try {
                Os.dup2(descriptors[0], STDIN_FILENO);
                Os.dup2(descriptors[1], STDOUT_FILENO);
                Os.dup2(descriptors[2], STDERR_FILENO);
                for (FileDescriptor fd: descriptors) {
                    IoUtils.closeQuietly(fd);
                }
                newStderr = System.err;
            } catch (ErrnoException ex) {
                Log.e(TAG, "Error reopening stdio", ex);
            }
        }

        if (parsedArgs.niceName != null) {
            //设置进程名
            Process.setArgV0(parsedArgs.niceName);
        }

        if (parsedArgs.invokeWith != null) {
            //据说这是用于检测进程内存泄露或溢出时场景而设计，后续还需要进一步分析。
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(),
                    pipeFd, parsedArgs.remainingArgs);
        } else {
            //执行目标类的main()方法 【见流程14】
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null);
        }
    }

### 14. zygoteInit

[-->RuntimeInit.java]

    public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {

        redirectLogStreams(); //重定向log输出

        commonInit(); // 通用的一些初始化【见流程14.1】
        nativeZygoteInit(); // zygote初始化 【见流程14.2】
        applicationInit(targetSdkVersion, argv, classLoader); // 应用初始化【见流程14.3】
    }

#### 14.1 commonInit

[-->RuntimeInit.java]

    private static final void commonInit() {
        // 设置默认的未捕捉异常处理方法
        Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());

        // 设置市区，中国时区为"Asia/Shanghai"
        TimezoneGetter.setInstance(new TimezoneGetter() {
            public String getId() {
                return SystemProperties.get("persist.sys.timezone");
            }
        });
        TimeZone.setDefault(null);

        //重置log配置
        LogManager.getLogManager().reset();
        new AndroidConfig();

        // 设置默认的HTTP User-agent格式,用于 HttpURLConnection。
        String userAgent = getDefaultUserAgent();
        System.setProperty("http.agent", userAgent);

        // 设置socket的tag，用于网络流量统计
        NetworkManagementSocketTagger.install();
    }

默认的HTTP User-agent格式，例如：

     "Dalvik/1.1.0 (Linux; U; Android 6.0.1；LenovoX3c70 Build/LMY47V)".

#### 14.2 nativeZygoteInit
nativeZygoteInit()所对应的jni方法如下：

[-->AndroidRuntime.cpp]

    static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
    {
        //此处的gCurRuntime为AppRuntime，是在AndroidRuntime.cpp中定义的
        gCurRuntime->onZygoteInit();
    }

##### 14.2.1 onZygoteInit
[-->app_main.cpp]

    virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        proc->startThreadPool(); //启动新binder线程
    }

- ProcessState::self():主要工作是调用open()打开/dev/binder驱动设备，再利用mmap()映射内核的地址空间，将Binder驱动的fd赋值ProcessState对象中的变量mDriverFD，用于交互操作。startThreadPool()是创建一个新的binder线程，不断进行talkWithDriver().
- startThreadPool(): 启动Binder线程池, 详见[进程的Binder线程池工作过程](http://gityuan.com/2016/10/29/binder-thread-pool/)


#### 14.3 applicationInit
[-->RuntimeInit.java]

    private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        //true代表应用程序退出时不调用AppRuntime.onExit()，否则会在退出前调用
        nativeSetExitWithoutCleanup(true);

        //设置虚拟机的内存利用率参数值为0.75
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args;
        try {
            args = new Arguments(argv); //解析参数
        } catch (IllegalArgumentException ex) {
            return;
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        //调用startClass的static方法 main() 【见流程15】
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }

此处args.startClass为"android.app.ActivityThread"。

### 15. invokeStaticMain

[-->RuntimeInit.java]

    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl = Class.forName(className, true, classLoader);

        Method m = cl.getMethod("main", new Class[] { String[].class });

        int modifiers = m.getModifiers();
        ...

        //通过抛出异常，回到ZygoteInit.main()。这样做好处是能清空栈帧，提高栈帧利用率。【见流程16】
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }

invokeStaticMain()方法中抛出的异常`MethodAndArgsCaller` caller，该方法的参数`m`是指main()方法, `argv`是指ActivityThread.
根据前面的【流程4】中可知，下一步进入caller.run()方法，也就是MethodAndArgsCaller.run()。

### 16. MethodAndArgsCaller

[-->ZygoteInit.java]

    public static class MethodAndArgsCaller extends Exception
            implements Runnable {

        public void run() {
            try {
                //根据传递过来的参数，此处反射调用ActivityThread.main()方法【见流程17】
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
    }

到此，总算是进入到了ActivityThread类的main()方法。

### 17. ActivityThread.main
[--> ActivityThread.java]

    public static void main(String[] args) {
        ...
        Environment.initForCurrentUser();
        ...
        Process.setArgV0("<pre-initialized>");
        //创建主线程looper
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        //attach到系统进程
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        //主线程进入循环状态
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

## 五. 总结

Process.start()方法是阻塞操作，等待直到进程创建完成并返回相应的新进程pid，才完成该方法。

当App第一次启动时或者启动远程Service，即AndroidManifest.xml文件中定义了process:remote属性时，都需要创建进程。比如当用户点击桌面的某个App图标，桌面本身是一个app（即Launcher App），那么Launcher所在进程便是这次创建新进程的发起进程，该通过binder发送消息给system_server进程，该进程承载着整个java framework的核心服务。system_server进程从Process.start开始，执行创建进程，流程图（以进程的视角）如下：

点击查看[大图](http://gityuan.com/images/android-process/process-create.jpg)

![process-create](/images/android-process/process-create.jpg)

上图中，`system_server`进程通过socket IPC通道向`zygote`进程通信，`zygote`在fork出新进程后由于fork**调用一次，返回两次**，即在zygote进程中调用一次，在zygote进程和子进程中各返回一次，从而能进入子进程来执行代码。该调用流程图的过程：

1. **system_server进程**（`即流程1~3`）：通过Process.start()方法发起创建新进程请求，会先收集各种新进程uid、gid、nice-name等相关的参数，然后通过socket通道发送给zygote进程；
2. **zygote进程**（`即流程4~12`）：接收到system_server进程发送过来的参数后封装成Arguments对象，图中绿色框forkAndSpecialize()方法是进程创建过程中最为核心的一个环节（[详见流程6](http://gityuan.com/2016/03/26/app-process-create/#forkandspecialize-1)），其具体工作是依次执行下面的3个方法：
    - preFork()：先停止Zygote的4个Daemon子线程（java堆内存整理线程、对线下引用队列线程、析构线程以及监控线程）的运行以及初始化gc堆；
    - nativeForkAndSpecialize()：调用linux的fork()出新进程，创建Java堆处理的线程池，重置gc性能数据，设置进程的信号处理函数，启动JDWP线程；
    - postForkCommon()：在启动之前被暂停的4个Daemon子线程。
3. **新进程**（`即流程13~15`）：进入handleChildProc()方法，设置进程名，打开binder驱动，启动新的binder线程；然后设置art虚拟机参数，再反射调用目标类的main()方法，即Activity.main()方法。

再之后的流程，如果是startActivity则将要进入Activity的onCreate/onStart/onResume等生命周期；如果是startService则将要进入Service的onCreate等生命周期。

system_server进程等待zygote返回进程创建完成(ZygoteConnection.handleParentProc), 一旦Zygote.forkAndSpecialize()方法执行完成, 那么分道扬镳, zygote告知system_server进程进程已创建,
而子进程继续执行后续的handleChildProc操作.

Tips: [小节11]RuntimeInit.java的方法nativeZygoteInit()会调用到onZygoteInit()，这个过程中有startThreadPool()创建Binder线程池。也就是说每个进程无论是否包含任何activity等组件，一定至少会包含一个Binder线程。
