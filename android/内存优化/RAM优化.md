# RAM优化定义
- 主要是降低运行时内存。它的目的如下：
	- 防止应用发生OOM。
	- 降低应用由于内存过大被LMK机制杀死的概率。
	- 避免不合理使用内存导致GC次数增多，从而导致应用发生卡顿。 
- RAM优化主要是从内存抖动，内存泄漏，内存溢出这三个方面进行优化
# 内存抖动
## 定义
- 内存抖动是由于短时间内有大量对象进出新生区导致的，它伴随着频繁的GC。 
gc会大量占用ui线程和cpu资源，会导致app整体卡顿
## 定位
- 一般使用Memory Profiler或CPU Profiler结合代码排查即可找到内存抖动出现的地方。
- 通常的技巧就是着重查看循环或频繁调用的地方。
## 常见案例解决
1. 字符串使用加号拼接：
	- 使用StringBuilder(线程不安全)替代。
	- 初始化时设置容量，减少StringBuilder的扩容。
2. 资源复用
	- 使用全局缓存池，以重用频繁申请和释放的对象。
	- 注意结束使用后，需要手动释放对象池中的对象。
3. 减少不合理的对象创建
	- ondraw、getView中对象的创建尽量进行复用。
	- 避免在循环中不断创建局部变量。
4. 使用合理的数据结构
	- 使用SparseArray类族来替代HashMap。
# 内存泄漏
## 内存泄漏定义
- Android系统虚拟机的垃圾回收是通过虚拟机GC机制来实现的。GC会选择一些还存活的对象作为内存遍历的根节点GC Roots，通过对GC Roots的可达性来判断是否需要回收。内存泄漏就是在当前应用周期内不再使用的对象被GC Roots引用，导致不能回收，使实际可使用内存变小。
## 常见场景
1. 资源性对象未关闭
	- 对于资源性对象不再使用时，应该立即调用它的close()函数，将其关闭，然后在置为null。
2. 注册对象未注销
3. 类的静态变量持有大数据对象
4. 非静态内部类的静态实例
	- 该实例的生命周期和应用一样长，这就导致该静态实例一直持有该Activity的引用，Activity的内存资源不能正常回收。
	- 解决方案：
		- 将内部类设为静态内部类或将内部类抽取来作为一个单例，如果需要使用Context，尽量使用Application Context，如果需要使用Activity Context，就记得用完后置空让GC可以回收，否则还是会内存泄漏。
5. Handler临时性内存泄漏
	- Message发出之后存储在MessageQueue中，在Message中存在一个target，它是Handler的一个引用，Message在Queue中存在的时间过长，就会导致Handler无法被回收。如果Handler是非静态的，则会导致Activity或者Service不会被回收。并且消息队列是在一个Looper线程中不断地轮询处理消息，当这个Activity退出时，消息队列中还有未处理的消息或者正在处理的消息，并且消息队列中的Message持有Handler实例的引用，Handler又持有Activity的引用，所以导致该Activity的内存资源无法及时回收，引发内存泄漏。
	- 解决方案：
		- 使用一个静态Handler内部类，然后对Handler持有的对象（一般是Activity）使用弱引用，这样在回收时，也可以回收Handler持有的对象。
		- 在Activity的Destroy或者Stop时，应该移除消息队列中的消息，避免Looper线程的消息队列中有待处理的消息需要处理。
		- 注意：AsyncTask内部也是Handler机制，同样存在内存泄漏风险，当其一般是临时性的。
6. 容器中的对象没清理造成的内存泄漏
7. WebView
	- WebView都存在内存泄漏的问题，在应用中只要使用一次WebView，内存就不会被释放掉。
	- 解决方案：
	- 为WebView开启一个独立的进程，使用AIDL与应用的主进程进行通信，WebView所在的进程可以根据业务的需要选择合适的时机进行销毁，达到正常释放内存的目的。
## 内存泄漏监测
### LeakCanary
#### 使用示例
- 首先在你项目app下的build.gradle中配置:

<pre>
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.2'
  releaseImplementation   'com.squareup.leakcanary:leakcanary-android-no-op:1.6.2'
  // 可选，如果你使用支持库的fragments的话
  debugImplementation   'com.squareup.leakcanary:leakcanary-support-fragment:1.6.2'
}
</pre>
- 然后在你的Application中配置:

<pre>
public class WanAndroidApp extends Application {

    private RefWatcher refWatcher;

    public static RefWatcher getRefWatcher(Context context) {
        WanAndroidApp application = (WanAndroidApp)     context.getApplicationContext();
        return application.refWatcher;
    }

    @Override public void onCreate() {
      super.onCreate();
      if (LeakCanary.isInAnalyzerProcess(this)) {
        // 1
        return;
      }
      // 2
      refWatcher = LeakCanary.install(this);
    }
}
</pre>
- 在注释1处，会首先判断当前进程是否是Leakcanary专门用于分析heap内存的而创建的那个进程，即HeapAnalyzerService所在的进程，如果是的话，则不进行Application中的初始化功能。如果是当前应用所处的主进程的话，则会执行注释2处的LeakCanary.install(this)进行LeakCanary的安装。只需这样简单的几行代码，我们就可以在应用中检测是否产生了内存泄露了。当然，这样使用只会检测Activity是否发生内存泄漏，如果要检测Fragment在执行完onDestroy()之后是否发生内存泄露的话，则需要在Fragment的onDestroy()方法中加上如下两行代码去监视当前的Fragment：
<pre>
RefWatcher refWatcher = WanAndroidApp.getRefWatcher(_mActivity);
refWatcher.watch(this);
</pre>
- 上面的RefWatcher其实就是一个引用观察者对象，是用于监测当前实例对象的引用状态的。从以上的分析可以了解到，核心代码就是LeakCanary.install(this)这行代码，接下来，就从这里出发将LeakCanary一步一步进行拆解。
#### 自定义处理结果
- 首先，继承DisplayLeakService实现一个自定义的监控处理Service，代码如下：
<pre>
public class LeakCnaryService extends DisplayLeakServcie {

    private final String TAG = “LeakCanaryService”；

    @Override
    protected void afterDefaultHandling(HeapDump heapDump， AnalysisResult result， String leakInfo) {
        ...
    }
}
</pre>
- 重写afterDefaultHanding方法，在其中处理需要的数据，三个参数的定义如下：

- heapDump：堆内存文件，可以拿到完成的hprof文件，以使用MAT分析。
- result：监控到的内存状态，如是否泄漏等。
- leakInfo：leak trace详细信息，除了内存泄漏对象，还有设备信息。
- 然后在install时，使用自定义的LeakCanaryService即可，代码如下：
<pre>
public class BaseApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        mRefWatcher = LeakCanary.install(this, LeakCanaryService.calss, AndroidExcludedRefs.createAppDefaults().build());
    }

    ...

}
</pre>
- 经过这样的处理，就可以在LeakCanaryService中实现自己的处理方式，如丰富的提示信息，把数据保存在本地、上传到服务器进行分析。

- 注意：LeakCanaryService需要在AndroidManifest中注册。
#### 使用AndroidProfiler的MEMORY工具：
- 运行程序，对每一个页面进行内存分析检查。首先，反复打开关闭页面5次，然后手动GC（点击Profile MEMORY左上角的垃圾桶图标），如果此时total内存还没有恢复到之前的数值，则可能发生了内存泄露。此时，再点击Profile MEMORY左上角的垃圾桶图标旁的heap dump按钮查看当前的内存堆栈情况，选择按包名查找，找到当前测试的Activity，如果引用了多个实例，则表明发生了内存泄露。
# 内存溢出
## 缓存（Lrucache）
## bitma采样率优化