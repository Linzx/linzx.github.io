---
layout: post
title:  "Android系统启动-zygote篇"
date:   2016-02-13 20:21:40
catalog:  true
tags:
    - android
    - 系统启动


---

> 基于Android 6.0的源码剖析， 分析Android启动过程的Zygote进程

    /frameworks/base/cmds/app_process/App_main.cpp
    /frameworks/base/core/jni/AndroidRuntime.cpp

    /frameworks/base/core/java/com/android/internal/os/
      - ZygoteInit.java
      - Zygote.java
      - ZygoteConnection.java
      
    /frameworks/base/core/java/android/net/LocalServerSocket.java
    /system/core/libutils/Threads.cpp

## 一. 概述

Zygote是由[init进程](http://gityuan.com/2016/02/05/android-init/)通过解析init.zygote.rc文件而创建的，zygote所对应的可执行程序app_process，所对应的源文件是App_main.cpp，进程名为zygote。

    service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
        class main
        socket zygote stream 660 root system
        onrestart write /sys/android_power/request_state wake
        onrestart write /sys/power/state on
        onrestart restart media
        onrestart restart netd


Zygote进程能够重启的地方: 

- servicemanager进程被杀; (onresart)
- surfaceflinger进程被杀; (onresart)
- Zygote进程自己被杀; (oneshot=false)
- system_server进程被杀; (waitpid)
        
从App_main()开始，Zygote启动过程的函数调用类大致流程如下：

![zygote_process](/images/boot/zygote/zygote_process.jpg)

## 二、Zygote启动过程

### 2.1 App_main.main
[-> App_main.cpp]

    int main(int argc, char* const argv[])
    {
        //传到的参数argv为“-Xzygote /system/bin --zygote --start-system-server”
        AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
        argc--; argv++; //忽略第一个参数

        int i;
        for (i = 0; i < argc; i++) {
            if (argv[i][0] != '-') {
                break;
            }
            if (argv[i][1] == '-' && argv[i][2] == 0) {
                ++i;
                break;
            }
            runtime.addOption(strdup(argv[i]));
        }
        //参数解析
        bool zygote = false;
        bool startSystemServer = false;
        bool application = false;
        String8 niceName;
        String8 className;
        ++i;
        while (i < argc) {
            const char* arg = argv[i++];
            if (strcmp(arg, "--zygote") == 0) {
                zygote = true;
                //对于64位系统nice_name为zygote64; 32位系统为zygote
                niceName = ZYGOTE_NICE_NAME;
            } else if (strcmp(arg, "--start-system-server") == 0) {
                startSystemServer = true;
            } else if (strcmp(arg, "--application") == 0) {
                application = true;
            } else if (strncmp(arg, "--nice-name=", 12) == 0) {
                niceName.setTo(arg + 12);
            } else if (strncmp(arg, "--", 2) != 0) {
                className.setTo(arg);
                break;
            } else {
                --i;
                break;
            }
        }
        Vector<String8> args;
        if (!className.isEmpty()) {
            // 运行application或tool程序
            args.add(application ? String8("application") : String8("tool"));
            runtime.setClassNameAndArgs(className, argc - i, argv + i);
        } else {
            //进入zygote模式，创建 /data/dalvik-cache路径
            maybeCreateDalvikCache();
            if (startSystemServer) {
                args.add(String8("start-system-server"));
            }
            char prop[PROP_VALUE_MAX];
            if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
                return 11;
            }
            String8 abiFlag("--abi-list=");
            abiFlag.append(prop);
            args.add(abiFlag);

            for (; i < argc; ++i) {
                args.add(String8(argv[i]));
            }
        }

        //设置进程名
        if (!niceName.isEmpty()) {
            runtime.setArgv0(niceName.string());
            set_process_name(niceName.string());
        }
        if (zygote) {
            // 启动AppRuntime 【见小节2.2】
            runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
        } else if (className) {
            runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
        } else {
            //没有指定类名或zygote，参数错误
            return 10;
        }
    }

## 2.2 start
[-> AndroidRuntime.cpp]

    void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
    {
        static const String8 startSystemServer("start-system-server");

        for (size_t i = 0; i < options.size(); ++i) {
            if (options[i] == startSystemServer) {
               const int LOG_BOOT_PROGRESS_START = 3000;
            }
        }
        const char* rootDir = getenv("ANDROID_ROOT");
        if (rootDir == NULL) {
            rootDir = "/system";
            if (!hasDir("/system")) {
                return;
            }
            setenv("ANDROID_ROOT", rootDir, 1);
        }
        JniInvocation jni_invocation;
        jni_invocation.Init(NULL);
        JNIEnv* env;
        // 虚拟机创建【见小节2.3】
        if (startVm(&mJavaVM, &env, zygote) != 0) {
            return;
        }
        onVmCreated(env);
        // JNI方法注册【见小节2.4】
        if (startReg(env) < 0) {
            return;
        }

        jclass stringClass;
        jobjectArray strArray;
        jstring classNameStr;

        //等价 strArray= new String[options.size() + 1];
        stringClass = env->FindClass("java/lang/String");
        strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);

        //等价 strArray[0] = "com.android.internal.os.ZygoteInit"
        classNameStr = env->NewStringUTF(className);
        env->SetObjectArrayElement(strArray, 0, classNameStr);

        //等价 strArray[1] = "start-system-server"；
        //    strArray[2] = "--abi-list=xxx"；
        //其中xxx为系统响应的cpu架构类型，比如arm64-v8a.
        for (size_t i = 0; i < options.size(); ++i) {
            jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
            env->SetObjectArrayElement(strArray, i + 1, optionsStr);
        }

        //将"com.android.internal.os.ZygoteInit"转换为"com/android/internal/os/ZygoteInit"
        char* slashClassName = toSlashClassName(className);
        jclass startClass = env->FindClass(slashClassName);
        if (startClass == NULL) {
            ...
        } else {
            jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
                "([Ljava/lang/String;)V");
            // 调用ZygoteInit.main()方法【见小节3.1】
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
        }
        //释放相应对象的内存空间
        free(slashClassName);
        mJavaVM->DetachCurrentThread();
        mJavaVM->DestroyJavaVM();
    }


### 2.3 startVm
[--> AndroidRuntime.cpp]

创建Java虚拟机方法的主要篇幅是关于虚拟机参数的设置，下面只列举部分在调试优化过程中常用参数。

    int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
    {
        // JNI检测功能，用于native层调用jni函数时进行常规检测，比较弱字符串格式是否符合要求，资源是否正确释放。该功能一般用于早期系统调试或手机Eng版，对于User版往往不会开启，引用该功能比较消耗系统CPU资源，降低系统性能。
        bool checkJni = false;
        property_get("dalvik.vm.checkjni", propBuf, "");
        if (strcmp(propBuf, "true") == 0) {
            checkJni = true;
        } else if (strcmp(propBuf, "false") != 0) {
            property_get("ro.kernel.android.checkjni", propBuf, "");
            if (propBuf[0] == '1') {
                checkJni = true;
            }
        }
        if (checkJni) {
            addOption("-Xcheck:jni");
        }

        //虚拟机产生的trace文件，主要用于分析系统问题，路径默认为/data/anr/traces.txt
        parseRuntimeOption("dalvik.vm.stack-trace-file", stackTraceFileBuf, "-Xstacktracefile:");

        //对于不同的软硬件环境，这些参数往往需要调整、优化，从而使系统达到最佳性能
        parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
        parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");
        parseRuntimeOption("dalvik.vm.heapgrowthlimit", heapgrowthlimitOptsBuf, "-XX:HeapGrowthLimit=");
        parseRuntimeOption("dalvik.vm.heapminfree", heapminfreeOptsBuf, "-XX:HeapMinFree=");
        parseRuntimeOption("dalvik.vm.heapmaxfree", heapmaxfreeOptsBuf, "-XX:HeapMaxFree=");
        parseRuntimeOption("dalvik.vm.heaptargetutilization",
                           heaptargetutilizationOptsBuf, "-XX:HeapTargetUtilization=");
        ...

        //preloaded-classes文件内容是由WritePreloadedClassFile.java生成的，
        //在ZygoteInit类中会预加载工作将其中的classes提前加载到内存，以提高系统性能
        if (!hasFile("/system/etc/preloaded-classes")) {
            return -1;
        }

        //初始化虚拟机
        if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
            ALOGE("JNI_CreateJavaVM failed\n");
            return -1;
        }
    }


### 2.4 startReg
[--> AndroidRuntime.cpp]

    int AndroidRuntime::startReg(JNIEnv* env)
    {
        //设置线程创建方法为javaCreateThreadEtc 【见小节2.4.1】
        androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

        env->PushLocalFrame(200);
        //进程NI方法的注册【见小节2.4.2】
        if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
            env->PopLocalFrame(NULL);
            return -1;
        }
        env->PopLocalFrame(NULL);
        return 0;
    }

#### 2.4.1 androidSetCreateThreadFunc
[-> Threads.cpp]

    void androidSetCreateThreadFunc(android_create_thread_fn func)
    {
        gCreateThreadFn = func;
    }

虚拟机启动后startReg()过程，会设置线程创建函数指针`gCreateThreadFn`指向`javaCreateThreadEtc`.

#### 2.4.2 register_jni_procs

    static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
    {
        for (size_t i = 0; i < count; i++) {
            //【见小节2.4.3】
            if (array[i].mProc(env) < 0) {
                return -1;
            }
        }
        return 0;
    }

#### 2.4.3 gRegJNI.mProc

    static const RegJNIRec gRegJNI[] = {
        REG_JNI(register_com_android_internal_os_RuntimeInit),
        REG_JNI(register_android_os_Binder)，
        ...
    }；

array[i]是指gRegJNI数组, 该数组有100多个成员。其中每一项成员都是通过**REG_JNI**宏定义的：

    #define REG_JNI(name)      { name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
    };

可见，调用`mProc`，就等价于调用其参数名所指向的函数。 例如REG_JNI(register_com_android_internal_os_RuntimeInit).mProc也就是指进入register_com_android_internal_os_RuntimeInit方法，接下来就继续以此为例来说明：


    int register_com_android_internal_os_RuntimeInit(JNIEnv* env)
    {
        return jniRegisterNativeMethods(env, "com/android/internal/os/RuntimeInit",
            gMethods, NELEM(gMethods));
    }

    //gMethods：java层方法名与jni层的方法的一一映射关系
    static JNINativeMethod gMethods[] = {
        { "nativeFinishInit", "()V",
            (void*) com_android_internal_os_RuntimeInit_nativeFinishInit },
        { "nativeZygoteInit", "()V",
            (void*) com_android_internal_os_RuntimeInit_nativeZygoteInit },
        { "nativeSetExitWithoutCleanup", "(Z)V",
            (void*) com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup },
    };

## 三. 进入Java层

前面[小节2.1]AndroidRuntime.start()执行到最后通过反射调用到ZygoteInit.main(),见下文:

### 3.1 ZygoteInit.main
[-->ZygoteInit.java]

    public static void main(String argv[]) {
        try {
            RuntimeInit.enableDdms(); //开启DDMS功能
            SamplingProfilerIntegration.start();
            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }
            ...

            registerZygoteSocket(socketName); //为Zygote注册socket【见小节3.2】
            preload(); // 预加载类和资源【见小节3.3】
            SamplingProfilerIntegration.writeZygoteSnapshot();
            gcAndFinalize(); //GC操作
            if (startSystemServer) {
                startSystemServer(abiList, socketName);//启动system_server【见小节3.4】
            }
            runSelectLoop(abiList); //进入循环模式【见小节3.5】
            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run(); //启动system_server中会讲到。
        } catch (RuntimeException ex) {
            closeServerSocket();
            throw ex;
        }
    }

在异常捕获后调用的方法caller.run()，会在后续的system_server文章会讲到。

### 3.2 registerZygoteSocket
[-->ZygoteInit.java]

    private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                ...
            }

            try {
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc); //设置文件描述符
                sServerSocket = new LocalServerSocket(fd); //创建Socket的本地服务端
            } catch (IOException ex) {
                ...
            }
        }
    }

### 3.3 preload
[-->ZygoteInit.java]

    static void preload() {
        //预加载位于/system/etc/preloaded-classes文件中的类
        preloadClasses();

        //预加载资源，包含drawable和color资源
        preloadResources();

        //预加载OpenGL
        preloadOpenGL();

        //通过System.loadLibrary()方法，
        //预加载"android","compiler_rt","jnigraphics"这3个共享库
        preloadSharedLibraries();

        //预加载  文本连接符资源
        preloadTextResources();

        //仅用于zygote进程，用于内存共享的进程
        WebViewFactory.prepareWebViewInZygote();
    }

执行Zygote进程的初始化,对于类加载，采用反射机制Class.forName()方法来加载。对于资源加载，主要是 com.android.internal.R.array.preloaded_drawables和com.android.internal.R.array.preloaded_color_state_lists，在应用程序中以com.android.internal.R.xxx开头的资源，便是此时由Zygote加载到内存的。

zygote进程内加载了preload()方法中的所有资源，当需要fork新进程时，采用copy on write技术，如下：

![zygote_fork](/images/boot/zygote/zygote_fork.jpg)

### 3.4 startSystemServer
[-->ZygoteInit.java]

    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_BLOCK_SUSPEND,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_RESOURCE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG
        );
        //参数准备
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };

        ZygoteConnection.Arguments parsedArgs = null;
        int pid;
        try {
            //用于解析参数，生成目标格式
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            // fork子进程，用于运行system_server
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        //进入子进程system_server
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }
            // 完成system_server进程剩余的工作
            handleSystemServerProcess(parsedArgs);
        }
        return true;
    }

准备参数并fork新进程，从上面可以看出system server进程参数信息为uid=1000,gid=1000,进程名为sytem_server，从zygote进程fork新进程后，需要关闭zygote原有的socket。另外，对于有两个zygote进程情况，需等待第2个zygote创建完成。更多详情见[Android系统启动-systemServer上篇](http://gityuan.com/2016/02/14/android-system-server/)。

### 3.5 runSelectLoop
[-->ZygoteInit.java]

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
                    // 则创建ZygoteConnection对象,并添加到fds。
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor()); //添加到fds.
                } else {
                    //i>0，则代表通过socket接收来自对端的数据，并执行相应操作【见小节3.6】
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i); //处理完则从fds中移除该文件描述符
                    }
                }
            }
        }
    }

Zygote采用高效的I/O多路复用机制，保证在没有客户端连接请求或数据处理时休眠，否则响应客户端的请求。

### 3.6 runOnce
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
            ...
            return true;
        }
        ...

        try {
            //将binder客户端传递过来的参数，解析成Arguments对象格式
            parsedArgs = new Arguments(args);
            ...
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
                //进入子进程流程
                handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
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

更多内容，见[理解Android进程创建流程](http://gityuan.com/2016/03/26/app-process-create/)

## 四、总结

Zygote启动过程的调用流程图：

![zygote_start](/images/boot/zygote/zygote_start.jpg)

1. 解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法；
2. 调用AndroidRuntime的startVM()方法创建虚拟机，再调用startReg()注册JNI函数；
3. 通过JNI方式调用ZygoteInit.main()，第一次进入Java世界；
4. registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求；
5. preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView，用于提高app启动效率；
6. zygote完毕大部分工作，接下来再通过startSystemServer()，fork得力帮手system_server进程，也是上层framework的运行载体。
7. zygote功成身退，调用runSelectLoop()，随时待命，当接收到请求创建新进程请求时立即唤醒并执行相应工作。

最后，介绍给通过cmd命令，来fork新进程来执行类中main方法的方式：(启动后进入RuntimeInit.main)

     app_process [可选参数] 命令所在路径 启动的类名 [可选参数]
