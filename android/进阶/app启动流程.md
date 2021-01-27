> 文章参考  https://mp.weixin.qq.com/s/AwksinkWU099bLap01sZeA
# 总体流程
## Android应用进程的启动可以总结成以下步骤：

- 点击Launcher桌面的App图标
- AMS发起socket请求
- Zygote进程接收请求并处理参数
- Zygote进程fork出应用进程，应用进程继承得到虚拟机实例
- 应用进程启动binder线程池、运行ActivityThread类的main函数、启动Looper循环

# 逐一分析
## AMS发送socket请求 
- 点击App图标后经过层层调用会来到ActivityStackSupervisor的startSpecificActivityLocked方法
<pre>
//ActivityStackSupervisor.java
final ActivityManagerService mService;

void startSpecificActivityLocked(...) {
    //查找Activity所在的进程，ProcessRecord是用来封装进程信息的数据结构
    ProcessRecord app = mService.getProcessRecordLocked(...);
    //如果进程已启动，并且binder句柄IApplicationThread也拿到了，那就直接启动Activity
    if (app != null && app.thread != null) {
        realStartActivityLocked(r, app, andResume, checkConfig);
        return;
    }
    //否则，让AMS启动进程
    mService.startProcessLocked(...);
}
</pre>
- app.thread并不是线程，而是一个binder句柄。应用进程使用AMS需要拿到AMS的句柄IActivityManager，而系统需要通知应用和管理应用的生命周期，所以也需要持有应用进程的binder句柄IApplicationThread。
- 也就是说，他们互相持有彼此的binder句柄，来实现双向通信。
- 那IApplicationThread句柄是怎么传给AMS的呢？Zygote进程收到socket请求后会处理请求参数，执行ActivityThread的入口函数main。
<pre>
//ActivityThread.java
public static void main(String[] args) {
    //创建主线程的looper
    Looper.prepareMainLooper();
    //ActivityThread并不是线程，只是普通的java对象
    ActivityThread thread = new ActivityThread();
    //告诉AMS，应用已经启动好了
    thread.attach(false);
    //运行looper，启动消息循环
    Looper.loop();
}

private void attach(boolean system) {
    //获取AMS的binder句柄IActivityManager
    final IActivityManager mgr = ActivityManager.getService();
    //告诉AMS应用进程已经启动，并传入应用进程自己的binder句柄IApplicationThread
    mgr.attachApplication(mAppThread);
}
</pre>

- 所以对于AMS来说：

- AMS向Zygote发起启动应用的socket请求，Zygote收到请求fork出进程，返回进程的pid给AMS；
应用进程启动好后，执行入口main函数，通过attachApplication方法告诉AMS已经启动，同时传入应用进程的binder句柄IApplicationThread。

- 完成这两步，应用进程的启动过程才算完成。
- 下面看AMS的startProcessLocked启动应用进程时都做了些什么。
<pre>
//ActivityManagerService.java
final ProcessRecord startProcessLocked(...){
    ProcessRecord app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
    //如果进程信息不为空，并且已经拿到了Zygote进程返回的应用进程pid
    //说明AMS已经请求过了，并且Zygote已经响应请求然后fork出进程了
    if (app != null && app.pid > 0) {
        //但是app.thread还是空，说明应用进程还没来得及注册自己的binder句柄给AMS
        //即此时进程正在启动，那就直接返回，避免重复创建
        if (app.thread == null) {
            return app;
        }
    }
    //调用重载方法
    startProcessLocked(...);
}
</pre>
- 之所以要判断app.thread，是为了避免当应用进程正在启动的时候，假如又有另一个组件需要启动，导致重复拉起（创建）应用进程。
- 继续看重载方法startProcessLocked：
<pre>
//ActivityManagerService.java
private final void startProcessLocked(...){
    //应用进程的主线程的类名
    if (entryPoint == null) entryPoint = "android.app.ActivityThread";
    ProcessStartResult startResult = Process.start(entryPoint, ...);
}

//Process.java
public static final ProcessStartResult start(...){
    return zygoteProcess.start(...);
}
</pre>
- 来到ZygoteProcess。
<pre>
//ZygoteProcess.java
public final Process.ProcessStartResult start(...){
    return startViaZygote(...);
}

private Process.ProcessStartResult startViaZygote(...){
    ArrayList<String> argsForZygote = new ArrayList<String>();
    //...处理各种参数
    return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
}
</pre>
- 其中：

- openZygoteSocketIfNeeded打开本地socket
- zygoteSendArgsAndGetResult发送请求参数，其中带上了ActivityThread类名
- return返回的数据结构ProcessStartResult中会有pid字段


- 注意：Zygote进程启动时已经创建好了虚拟机实例，所以由他fork出的应用进程可以直接继承过来用而无需创建。

- 下面来看Zygote是如何处理socket请求的。

/   Zygote处理socket请求   /

- 从 图解Android系统的启动 一文可知，在ZygoteInit的main函数中，会创建服务端socket。
<pre>
//ZygoteInit.java
public static void main(String argv[]) {
    //Server类，封装了socket
    ZygoteServer zygoteServer = new ZygoteServer();
    //创建服务端socket，名字为socketName即zygote
    zygoteServer.registerServerSocket(socketName);
    //进入死循环，等待AMS发请求过来
    zygoteServer.runSelectLoop(abiList);
}
</pre>
- 看到ZygoteServer。
<pre>
//ZygoteServer.java
void registerServerSocket(String socketName) {
    int fileDesc;
    //socket真正的名字被加了个前缀，即 "ANDROID_SOCKET_" + "zygote"
    final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;

    String env = System.getenv(fullSocketName);
    fileDesc = Integer.parseInt(env);

    //创建文件描述符fd
    FileDescriptor fd = new FileDescriptor();
    fd.setInt$(fileDesc);
    //创建LocalServerSocket对象
    mServerSocket = new LocalServerSocket(fd);
}

void runSelectLoop(String abiList){
    //进入死循环
    while (true) {
        for (int i = pollFds.length - 1; i >= 0; --i) {
            if (i == 0) {
                //...
            } else {
                //得到一个连接对象ZygoteConnection，调用他的runOnce
                boolean done = peers.get(i).runOnce(this);
            }
        }
    }
}
</pre>
- 来到ZygoteConnection的runOnce。
<pre>
//ZygoteConnection.java
boolean runOnce(ZygoteServer zygoteServer){
    //读取socket请求的参数列表
    String args[] = readArgumentList();
    //创建应用进程
    int pid = Zygote.forkAndSpecialize(...);
    if (pid == 0) {
        //如果是应用进程(Zygote fork出来的子进程)，处理请求参数
        handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
        return true;
    } else {
        return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
    }
}
</pre>
- handleChildProc方法调用了ZygoteInit的zygoteInit方法，里边主要做了3件事：

- 启动binder线程池（后面分析）
- 读取请求参数拿到ActivityThread类并执行他的main函数，执行thread.attach告知AMS并回传自己的binder句柄
- 执行Looper.loop()启动消息循环（代码前面有）

- 这样应用进程就启动起来了。梳理一下：

- 下面看下binder线程池是怎么启动的。

/   启动binder线程池   /

- Zygote的跨进程通信没有使用binder，而是socket，所以应用进程的binder机制不是继承而来，而是进程创建后自己启动的。

- 前边可知，Zygote收到socket请求后会得到一个ZygoteConnection，他的runOnce会调用handleChildProc。
<pre>
//ZygoteConnection.java
private void handleChildProc(...){
    ZygoteInit.zygoteInit(...);
}

//ZygoteInit.java
public static final void zygoteInit(...){
    RuntimeInit.commonInit();
    //进入native层
    ZygoteInit.nativeZygoteInit();
    RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
}
</pre>
- 来到AndroidRuntime.cpp：
<pre>
//AndroidRuntime.cpp
static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz){
    gCurRuntime->onZygoteInit();
}
</pre>
来到app_main.cpp：
<pre>
//app_main.cpp
virtual void onZygoteInit()
{
    //获取单例
    sp<ProcessState> proc = ProcessState::self();
    //在这里启动了binder线程池
    proc->startThreadPool();
}
</pre>
- 看下ProcessState.cpp：
<pre>
//ProcessState.cpp
sp<ProcessState> ProcessState::self()
{
    //单例模式，返回ProcessState对象
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}

//ProcessState构造函数
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
        , mDriverFD(open_driver(driver)) //打开binder驱动
        ,//...
{
    if (mDriverFD >= 0) {
        //mmap是一种内存映射文件的方法，把mDriverFD映射到当前的内存空间
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, 
                        MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
    }
}

//启动了binder线程池
void ProcessState::startThreadPool()
{
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}

void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        //创建线程名字"Binder:${pid}_${自增数字}"
        String8 name = makeBinderThreadName();
        sp<Thread> t = new PoolThread(isMain);
        //运行binder线程
        t->run(name.string());
    }
}
</pre>
- ProcessState有两个宏定义值得注意一下：
<pre>
//ProcessState.cpp
//一次Binder通信最大可以传输的大小是 1MB-4KB*2
#define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)
//binder驱动的文件描述符fd被限制了最大线程数15
#define DEFAULT_MAX_BINDER_THREADS 15
</pre>
- 我们看下binder线程PoolThread长啥样：
<pre>
class PoolThread : public Thread
{
public:
    explicit PoolThread(bool isMain)
        : mIsMain(isMain){}
protected:
    virtual bool threadLoop()
    {    //把binder线程注册进binder驱动程序的线程池中
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }

    const bool mIsMain;
};
</pre>
- 来到IPCThreadState.cpp：
<pre>
//IPCThreadState.cpp
void IPCThreadState::joinThreadPool(bool isMain)
{
    //向binder驱动写数据：进入死循环
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    status_t result;
    do {
        //进入死循环，等待指令的到来
        result = getAndExecuteCommand();
    } while (result != -ECONNREFUSED && result != -EBADF);
    //向binder驱动写数据：退出死循环
    mOut.writeInt32(BC_EXIT_LOOPER);
}

status_t IPCThreadState::getAndExecuteCommand()
{
    //从binder驱动读数据，得到指令
    cmd = mIn.readInt32();
    //执行指令
    result = executeCommand(cmd);
    return result;
}
</pre>
梳理一下binder的启动过程：

- 打开binder驱动
- 映射内存，分配缓冲区
- 运行binder线程，进入死循环，等待指令
