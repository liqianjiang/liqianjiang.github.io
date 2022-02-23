---
layout:     post
title:      native crash采集
subtitle:   native开发
date:       2022-02-23
author:     River
header-img: img/native-crash/post-bg-ko.jpg
catalog: true
tags:
    - 用户态 内核态
    - Android
--- 

# 前言
端上crash监控非常基础，也很常见；小公司一般用的bugly、友盟等三方免费服务，到公司开发APM的时候，
基础技术决定自研线上crash采集监控，Android端侧apm开发由我负责；整个过程内心是很崩溃的。


### 1、native crash的定义

![naive crash的定义](/img/native-crash/native_crash_define.png)

每个Android App再运行时是一个Linux进程，Linux进程运行时，空间是在用户态。我们所能处理的crash，
其实也就是用户态的crash，包括两类，一是Android Runtime捕获的Exception或Error，这是我们日常
业务开发直接接触的，我们称之为java crash。而是Runtime自身,Native Libraries,HAL产生的crash，
这是Linux内核通知app进程的，我们称之为native crash。



### 2、捕获原理

![naive crash的捕获原理](/img/native-crash/native_pipline.png)

#### 流程分析
Android定制的Linux Bionic内核对于用户态进程crash的处理流程是这样的;<br>
(1) 无论进程运行过程中发生什么异常，都是cpu先报错，同时向内核发出硬中断;<br>
(2) 内核会将所有类型的硬中断封装中一组信号量，并将代表错误的信号加入该进程的信号队列中，同时还会向它发出软中断。
注意: 此时信号还只是在队列中，对于进程来说还是未知的。<br>
(3) 进程收到中断后，用户态进入内核态，进行中断处理，这里的核心任务就是检测信号队列；<br>
(4) 进程发现有新的信号，于是将开始信号处理。因为信号处理的函数是运行在用户态上的，所以内核会先将当前内核栈的内容
拷贝到用户栈上，同时修改指令寄存器，指向信号处理函数。<br>
(5) 进程返回用户态，执行相应的信号处理函数；<br>
(6) 信号处理函数执行完，进程返回内核态，检查其他未处理的信号。如果所有信号都处理完成，将把内核栈从用户态的备份拷贝回来，
同时恢复指令寄存器，指向中断前运行的位子。<br>
(7) 最后进程返回用户态继续执行。<br>

从上面的分析可以看出，crash捕获的核心就在用户态的信号处理程序，这是我们实际开发中唯一可以对crash做出处理的地方。

### 3、native crash的采集

3.1、信号注册
Linux里信号处理，我们用sigaction()函数来实现。

```
#include <signal.h>

int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
```
其中，signum是信号，act是对该信号的处理（如果为NULL，则为缺省处理），oldact是旧的处理。

对于我们感兴趣的信号，会为它们注册一个统一的用于dump crash的处理。

```
// 仅供demo，勿当真
const int targetSignals[] = {
    SIGABRT, SIGFPE, SIGSEGV, SIGPIPE, SIGILL, SIGBUS, SIGSTKFLT, SIGTRAP
};
const int targetSignalsCount = sizeof(targetSignals) / sizeof(targetSignals[0]);
struct sigaction old_handlers[targetSignalsCount];

int setNativeCrashHandler(JNIEnv *env) {
    struct sigaction* handler = (struct sigaction*)malloc(sizeof(struct sigaction));
    if (!handler) {
        return -1;
    }

    memset(handler, 0, sizeof(struct sigaction));
    handler->sa_sigaction = diting_crash_handler;
    handler->sa_flags = SA_RESETHAND;

    for (int i = 0; i < targetSignalsCount; i++) {
        sigaction(targetSignals[i], handler, &old_handlers[i]);
    }

    return 0;
}

void diting_crash_handler(int signal, siginfo_t *info, void *context) {
    // TODO
}

```

3.2、Dump crash堆栈（diting_crash_handler）的实现

考虑到信号处理程序中的诸多限制，一般采用fork一个子进程，再在子进程中完成dump堆栈的方法。
注意，Linux里，fork创建的子进程与父进程是共享地址空间的。

![naive crash的堆栈dump](/img/native-crash/native_crash_dump.png)

ptrace(PTRACE_ATTACH,pid)会让目标进程$pid进入Sleeping状态，直到PTRACE_DETACH解除。
在中间的时间段里，我们可以获取目标进程$pid的各种状态信息，包括堆栈、寄存器、线程。
注意，自定义信号处理完成后，还要恢复调用信号的默认处理，切勿改变默认的处理流程。
例如Android里的Java NPE，底层就是通过SIGSEGV实现的。

### 4、native crash的符号化

4.1、符号表的生成

AndroidJNI开发已基本使用cmake，cmake编译产物有两类，libs下不带符号表的.so和objs下带符号表的.so。
我们只需将不带符号表的.so用于生产环境发布，将对应的带符号表的.so用于符号化即可。

可以使用Android内置的解析库corkscrew（< 5.0）或unwind（>= 5.0）。

### 5、native crash的聚合

聚合的关键指标有这么几个：App版本号、信号值和native堆栈。
前两个好理解，native堆栈如何聚合？我们取native堆栈的前N行，去除行号，作为聚合依据。

```
int HELLO_LINE = L1;    // 堆栈中哈啰研发团队代码的第一行
int MAX_LINE = L2;      // 堆栈最大行
if (HELLO_LINE < 0) {   // 不存在
    N = MAX_LINE;
} else {
    N = HELLO_LINE;
}
```

### 6、灰度计划

因牵涉稳定性基准指标，要求使用独立版本灰度

上线首日T，流量10%；对比友盟上native crash比例上升30%中止灰度；

上线T+3，流量30%；对比友盟上native crash比例上升60%中止灰度；

上线T+7日，流量放大到100%。


### 7、实际开发

阶段一: 开始尝试了大名鼎鼎的breakpad，解决一些编译错误后能够跑的起来；
初始化的时候传入一个SD卡的文件目录，breakpad会carsh的堆栈直接写入文件:

```
 NativeMonitor.init(new File(getExternalCacheDir(), "xxxx").getAbsolutePath());
 
```

缺点是代码量巨大，包体积增加明显;二次开发定制行极差，调试两天后放弃。

阶段二: 参考部分xcrash的代码以及网上的其他资料，磕磕碰碰总算完成。

核心实现分为两个阶段，监听信号量，堆栈回溯；<br>


在头文件定义需要捕获的信号以及捕获后恢复处理函数的:<br>

```
const int exceptionSignals[] = {SIGSEGV, SIGABRT, SIGFPE, SIGILL, SIGBUS, SIGTRAP};
const int exceptionSignalsNumber = sizeof(exceptionSignals)/ sizeof(exceptionSignals[0]);

// 系统值65，数组赋值在installSignalHandlers函数的sigaction方法调用处
static struct sigaction oldHandlers[NSIG];
```

#### 7.1 cpp文件注册新号函数

```
bool installSignalHandlers() {
    //保存原来的信号处理
    for (int i = 0; i < exceptionSignalsNumber; i++) {
        // sigaction是系统函数，对应的头文件#include <signal.h>
        // 指向结构体sigaction的一个实例的指针，该实例指定了对特定信号的处理，这里设置为空，进程会执行默认处理。
        if (sigaction(exceptionSignals[i], NULL, &oldHandlers[exceptionSignals[i]]) == -1) {
            return false;
        }
    }
    初始化一个信号处理对象
    struct sigaction sa{};
    memset(&sa, 0, sizeof(sa));
    //SA_ONSTACK|SA_SIGINFO表示不同堆栈的处理可将参数传递下去
    sa.sa_flags = SA_ONSTACK|SA_SIGINFO;
    // 核心的信号注册就在这里指定信号处理的回调函数
    sa.sa_sigaction = signalPass;
    //处理当前信号量的时候不考虑其他的
    for (int i = 0; i < exceptionSignalsNumber; ++i) {
        //阻塞其他信号的
        sigaddset(&sa.sa_mask, exceptionSignals[i]);
    }
    return true;
}


void signalPass(int code, siginfo_t *si, void *sc) {
    LOGE("监听到了native异常");
    // 非信号方式防止死锁，SIG_DFL表示默认的处理方式
    signal(code, SIG_DFL);
    signal(SIGALRM, SIG_DFL);
    // 解析栈信息，回调给 java 层，上报到后台或者保存本地文件
    notifyCaughtSignal(code, si, sc);
    // 给系统原来默认的处理，否则就会进入死循环
    oldHandlers[code].sa_sigaction(code, si, sc);
}

```

解析堆栈比信号注册复杂不少，一步步分析

#### 7.2  获取完整的上报数据
```
// copyInfo2Context用来，后面是唤醒线程
void notifyCaughtSignal(int code, siginfo_t *si, void *sc) {
    copyInfo2Context(code, si, sc);
    pthread_mutex_lock(&signalLock);
    pthread_cond_signal(&signalCond);
    pthread_mutex_unlock(&signalLock);
}
// 定义native-crash的结构体
typedef struct native_handler_context_struct {
    int code;
    siginfo_t *si;
    void *sc;
    pid_t pid;
    pid_t tid;
    const char *processName;
    const char *threadName;
    int frame_size;
    uintptr_t frames[BACKTRACE_FRAMES_MAX];
} native_handler_context;

// 核心系统方法_Unwind_GetIP使用#include <unwind.h>
_Unwind_Reason_Code unwind_callback(struct _Unwind_Context *context, void *arg) {
    native_handler_context *const s = static_cast<native_handler_context *const>(arg);
    //pc是每个堆栈的栈顶信息
    const uintptr_t pc = _Unwind_GetIP(context);
    if (pc != 0x0) {
        // 把 pc 值保存到 native_handler_context
        s->frames[s->frame_size++] = pc;
    }
    if (s->frame_size == BACKTRACE_FRAMES_MAX) {
        return _URC_END_OF_STACK;
    } else {
        return _URC_NO_REASON;
    }
}

// _Unwind_Backtrace使用#include <unwind.h>获取native的trace信息
void saveCaughtSignal(int code, siginfo_t *si, void *sc) {
    handlerContext->code = code;
    handlerContext->si = si;
    handlerContext->sc = sc;
    handlerContext->pid = getpid();
    handlerContext->tid = gettid();
    handlerContext->processName = getProcessName(handlerContext->pid);
    if (handlerContext->pid == handlerContext->tid) {
        handlerContext->threadName = "main";
    } else {
        handlerContext->threadName = getThreadName(handlerContext->tid);
    }
    handlerContext->frame_size = 0;
    //捕获c/c++的堆栈信息
    _Unwind_Backtrace(unwind_callback, handlerContext);
}

```

#### 7.3 把数据回调给java的回调

``` 
    public interface CrashListener {
        void onCrash(String threadName, Error error);
    }
    // Java的native函数初始化
    native void nativeCrashInit(CrashListener callback); 
    
    Java_com_xxx_crash_NativeCrashMonitor_nativeCrashInit(JNIEnv *env,
                                                           jobject nativeCrashMonitor,
                                                           jobject callback) {
    ...
    
    callback = env->NewGlobalRef(callback);
    jclass nativeCrashMonitorClass = env->GetObjectClass(nativeCrashMonitor);
    nativeCrashMonitorClass = (jclass) env->NewGlobalRef(nativeCrashMonitorClass);
    auto *jniBridge = new JNIBridge(javaVm, callback, nativeCrashMonitorClass);                          
    
    // threadCrashMonitor是函数指针，jniBridge是该函数的参数
    int ret = pthread_create(&pthread, NULL, threadCrashMonitor, jniBridge);
    if (ret < 0) {
        LOGE("%s", "pthread_create error");
    }
    ...
                                             
    }
    
    // threadCrashMonitor函数指针定义如下
    void *threadCrashMonitor(void *argv) {
    JNIBridge *jniBridge = static_cast<JNIBridge *>(argv);

        while (true) {
        //等待信号处理函数唤醒
        waitForSignal();

        //抛给java
        jniBridge->throwException2Java(handlerContext);
        }
    }
    
    //等待信号
    void waitForSignal() {
        pthread_mutex_lock(&signalLock);
        LOGE("waitForSignal start.");
        pthread_cond_wait(&signalCond, &signalLock);
        LOGE("waitForSignal finish.");
        pthread_mutex_unlock(&signalLock);
    }
    
    
    // dladdr是系统函数，用来获取pc加载的内存的起始地址
    void JNIBridge::throwException2Java(native_handler_context *handlerContext) {
    
    ...
    string result;
    for (int index = 0; index < frame_size; ++index) {
        uintptr_t pc = handlerContext->frames[index];
        //获取到加载的内存的起始地址
        Dl_info stack_info;
        void *const addr = (void *) pc;
        if (dladdr(addr, &stack_info) != 0 && stack_info.dli_fname != NULL) {
            if (stack_info.dli_fbase == 0) {// 非法地址
                result += "  <unknown>";
            } else if (stack_info.dli_fname) { // dli_fname名字不为空
                std::string so_name = std::string(stack_info.dli_fname);
                result += "  " + so_name;
            } else {
                result += android::base::StringPrintf("  <anonymous:%" PRIx64 ">",
                                                      (uint64_t) stack_info.dli_fbase);
            }
            
           ...
                
           result += ')';
          
           result += '\n';
        }
    }
            
            ...
            jclass crashClass = env->GetObjectClass(this->callbackObj);
            jmethodID crashMethod = env->GetMethodID(crashClass, "onCrash",
                                             "(Ljava/lang/String;Ljava/lang/Error;)V");
                                             
            jclass jErrorClass = env->FindClass("java/lang/Error");
            jmethodID jErrorInitMethod = env->GetMethodID(jErrorClass, "<init>", "(Ljava/lang/String;)V");
            jstring errorMessage = env->NewStringUTF(result.c_str());
            //callbackObj就是Java层传入的callback对象
            jobject errorObject = env->NewObject(jErrorClass, jErrorInitMethod, errorMessage);
            env->CallVoidMethod(this->callbackObj, crashMethod, jThreadName, errorObject);
            ...
    }  
    
```

#### 7.4 典型上下文的获取

```
// 通过读文件的方式，获取线程名称
extern const char *getThreadName(pid_t tid) {
    if (tid <= 1) {
        return NULL;
    }
    char *path = (char *) calloc(1, PATH_MAX);
    char *line = (char *) calloc(1, THREAD_NAME_LENGTH);
    snprintf(path, PATH_MAX, "/proc/%d/comm", tid);
    FILE *commFile = NULL;
    if (commFile = fopen(path, "r")) {
        // 指定最大读取的字符串的个数THREAD_NAME_LENGTH
        fgets(line, THREAD_NAME_LENGTH, commFile);
        fclose(commFile);
    }
    if (line) {
        int length = strlen(line);
        if (line[length - 1] == '\n') {
            line[length - 1] = '\0';
        }
    }
    free(path);
    return line;
}

// 通过读文件的方式，获取进程名称

extern const char *getProcessName(pid_t pid) {
    // 读一个文件
    if (pid <= 1) {
        return NULL;
    }
    char *path = (char *) calloc(1, PATH_MAX);
    char *line = (char *) calloc(1, PROCESS_NAME_LENGTH);
    snprintf(path, PATH_MAX, "/proc/%d/cmdline", pid);
    FILE *cmdFile = NULL;
    if (cmdFile = fopen(path, "r")) {
        fgets(line, PROCESS_NAME_LENGTH, cmdFile);
        fclose(cmdFile);
    }
    LOGD("line: %s",line);
    if (line) {
        int length = strlen(line);
        if (line[length - 1] == '\n') {
            line[length - 1] = '\0';
        }
    }
    LOGD("line: %s",line);
    free(path);
    return line;
}

```

### 总结

1. 有时候编写技术方案跟实际动手开发有明显的距离，有合适的开源框架站在前人的肩膀上不失为一种合适的办法；
2. 实际开发还是在开源示例的基础上改造，写过一次不再恐惧ndk的开发；
3. 纸上得来终觉浅，绝知此事要躬行。



