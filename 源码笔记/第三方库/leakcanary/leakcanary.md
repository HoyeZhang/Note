# 探究问题
- leakcancry是如何捕获内存泄露的
- 获取内存泄露时机（IdleHandler）
#基本原理
- 主要是在Activity的&onDestroy方法中，手动调用 GC，然后利用ReferenceQueue+WeakReference，来判断是否有释放不掉的引用，然后结合dump memory的hpof文件, 用HaHa分析出泄漏地方。
源码分析
- LeakCanary集成很方便，只要几行代码，所以可以从入口跟踪代码，分析原理
<pre>
                if (!LeakCanary.isInAnalyzerProcess(WeiboApplication.this)) {
                    LeakCanary.install(WeiboApplication.this);
                }
                
                public static RefWatcher install(Application application) {
                      return ((AndroidRefWatcherBuilder)refWatcher(application)
                      .listenerServiceClass(DisplayLeakService.class).excludedRefs(AndroidExcludedRefs.createAppDefaults().build()))//配置监听器及分析数据格式
                      .buildAndInstall();
               }
</pre>
- 复制代码从这里可看出，LeakCanary会单独开一进程，用来执行分析任务，和监听任务分开处理。
方法install中主要是构造来一个RefWatcher，
<pre>
   public RefWatcher buildAndInstall() {
        RefWatcher refWatcher = this.build();
        if(refWatcher != RefWatcher.DISABLED) {
            LeakCanary.enableDisplayLeakActivity(this.context);
            ActivityRefWatcher.install((Application)this.context, refWatcher);
        }

        return refWatcher;
    }
    
    public static void install(Application application, RefWatcher refWatcher) {
        (new ActivityRefWatcher(application, refWatcher)).watchActivities();
    }
    
    private final ActivityLifecycleCallbacks lifecycleCallbacks = new ActivityLifecycleCallbacks() {
        public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        }
        public void onActivityStarted(Activity activity) {}
        public void onActivityResumed(Activity activity) {}
        public void onActivityPaused(Activity activity) {}
        public void onActivityStopped(Activity activity) { }
        public void onActivitySaveInstanceState(Activity activity, Bundle outState) {}

        public void onActivityDestroyed(Activity activity) {
            ActivityRefWatcher.this.onActivityDestroyed(activity);
        }
    };
    
    void onActivityDestroyed(Activity activity) {
        this.refWatcher.watch(activity);
    }
</pre>
- 具体监听的原理在于 Application 的registerActivityLifecycleCallbacks方法，该方法可以对应用内所有 Activity 的生命周期做监听, LeakCanary只监听了Destroy方法。
在每个Activity的OnDestroy()方法中都会回调refWatcher.watch()方法，那我们找到的RefWatcher的实现类，看看具体做什么。
<pre>
 public void watch(Object watchedReference, String referenceName) {
        if(this != DISABLED) {
            Preconditions.checkNotNull(watchedReference, "watchedReference");
            Preconditions.checkNotNull(referenceName, "referenceName");
            long watchStartNanoTime = System.nanoTime();
            String key = UUID.randomUUID().toString();//保证key的唯一性
            this.retainedKeys.add(key);
            KeyedWeakReference reference = new KeyedWeakReference(watchedReference, key, referenceName, this.queue);
            this.ensureGoneAsync(watchStartNanoTime, reference);
        }
    }
    
    
  final class KeyedWeakReference extends WeakReference<Object> {
    public final String key;
    public final String name;

    KeyedWeakReference(Object referent, String key, String name, ReferenceQueue<Object> referenceQueue) {//ReferenceQueue类监听回收情况
        super(Preconditions.checkNotNull(referent, "referent"), (ReferenceQueue)Preconditions.checkNotNull(referenceQueue, "referenceQueue"));
        this.key = (String)Preconditions.checkNotNull(key, "key");
        this.name = (String)Preconditions.checkNotNull(name, "name");
    }
  }

    private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
        this.watchExecutor.execute(new Retryable() {
            public Result run() {
                return RefWatcher.this.ensureGone(reference, watchStartNanoTime);
            }
        });
    }
</pre>
- KeyedWeakReference是WeakReference类的子类，用了 
- KeyedWeakReference(referent,  key,  name, ReferenceQueue )的构造方法，将监听的对象（activity）引用传递进来，并且New出一个ReferenceQueue来监听GC后 的回收情况。
以下代码ensureGone()方法就是LeakCanary进行检测回收的核心代码：
<pre>
    Result ensureGone(KeyedWeakReference reference, long watchStartNanoTime) {
        long gcStartNanoTime = System.nanoTime();
        long watchDurationMs = TimeUnit.NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
        this.removeWeaklyReachableReferences();//先将引用尝试从队列中poll出来
        if(this.debuggerControl.isDebuggerAttached()) {//规避调试模式
            return Result.RETRY;
        } else if(this.gone(reference)) {//检测是否已经回收
            return Result.DONE;
        } else {
        //如果没有被回收，则手动GC
            this.gcTrigger.runGc();//手动GC方法
            this.removeWeaklyReachableReferences();//再次尝试poll，检测是否被回收
            if(!this.gone(reference)) {
    			// 还没有被回收，则dump堆信息，调起分析进程进行分析
                long startDumpHeap = System.nanoTime();
                long gcDurationMs = TimeUnit.NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
                File heapDumpFile = this.heapDumper.dumpHeap();
                if(heapDumpFile == HeapDumper.RETRY_LATER) {
                    return Result.RETRY;//需要重试
                }

                long heapDumpDurationMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
                this.heapdumpListener.analyze(new HeapDump(heapDumpFile, reference.key, reference.name, this.excludedRefs, watchDurationMs, gcDurationMs, heapDumpDurationMs));
            }

            return Result.DONE;
        }
    }

    private boolean gone(KeyedWeakReference reference) {
        return !this.retainedKeys.contains(reference.key);
    }

    private void removeWeaklyReachableReferences() {
        KeyedWeakReference ref;
        while((ref = (KeyedWeakReference)this.queue.poll()) != null) {
            this.retainedKeys.remove(ref.key);
        }
    }
</pre>
- 方法ensureGone中通过检测referenceQueue队列中的引用情况，来判断回收情况，通过手动GC来进一步确认回收情况。
- 整个过程肯定是个耗时卡UI的，整个过程会在WatchExecutor中执行的，那WatchExecutor又是在哪里执行的呢？
- LeakCanary已经利用Looper机制做了一定优化，利用主线程空闲的时候执行检测任务，这里找到WatchExecutor的实现类，研究下原理：
<pre>
public final class AndroidWatchExecutor implements WatchExecutor {
    static final String LEAK_CANARY_THREAD_NAME = "LeakCanary-Heap-Dump";
    private final Handler mainHandler = new Handler(Looper.getMainLooper());
    private final Handler backgroundHandler;
    private final long initialDelayMillis;
    private final long maxBackoffFactor;

    public AndroidWatchExecutor(long initialDelayMillis) {
        HandlerThread handlerThread = new HandlerThread("LeakCanary-Heap-Dump");
        handlerThread.start();
        this.backgroundHandler = new Handler(handlerThread.getLooper());
        this.initialDelayMillis = initialDelayMillis;
        this.maxBackoffFactor = 9223372036854775807L / initialDelayMillis;
    }

    public void execute(Retryable retryable) {
        if(Looper.getMainLooper().getThread() == Thread.currentThread()) {
            this.waitForIdle(retryable, 0);//需要在主线程中检测
        } else {
            this.postWaitForIdle(retryable, 0);//post到主线程
        }

    }

    void postWaitForIdle(final Retryable retryable, final int failedAttempts) {
        this.mainHandler.post(new Runnable() {
            public void run() {
                AndroidWatchExecutor.this.waitForIdle(retryable, failedAttempts);
            }
        });
    }

    void waitForIdle(final Retryable retryable, final int failedAttempts) {
        Looper.myQueue().addIdleHandler(new IdleHandler() {
            public boolean queueIdle() {
                AndroidWatchExecutor.this.postToBackgroundWithDelay(retryable, failedAttempts);//切换到子线程
                return false;
            }
        });
    }

    void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
        long exponentialBackoffFactor = (long)Math.min(Math.pow(2.0D, (double)failedAttempts), (double)this.maxBackoffFactor);
        long delayMillis = this.initialDelayMillis * exponentialBackoffFactor;
        this.backgroundHandler.postDelayed(new Runnable() {
            public void run() {
                Result result = retryable.run();//RefWatcher.this.ensureGone(reference, watchStartNanoTime)执行
                if(result == Result.RETRY) {
                    AndroidWatchExecutor.this.postWaitForIdle(retryable, failedAttempts + 1);
                }

            }
        }, delayMillis);
    }
}
</pre>
- 这里用到了Handler相关知识，Looper中的MessageQueue有个mIdleHandlers队列，在获取下个要执行的Message时，如果没有发现可执行的下个Msg，就会回调queueIdle()方法。
<pre>
    Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
           		···
           		···//省略部分消息查找代码
           		
           		if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        ···
                        
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

           		
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

                if (!keep) {//返回false，则从队列移除，下次空闲不会调用。
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
</pre>
- 其中的MessageQueue中加入一个IdleHandler，当线程空闲时，就会去调用*queueIdle()*函数，如果返回值为True，那么后续空闲时会继续的调用此函数，否则不再调用；
知识点
1，用ActivityLifecycleCallbacks接口来检测Activity生命周期
2，WeakReference + ReferenceQueue 来监听对象回收情况
3，Apolication中可通过processName判断是否是任务执行进程
4，MessageQueue中加入一个IdleHandler来得到主线程空闲回调
5，LeakCanary检测只针对Activiy里的相关对象。其他类无法使用，还得用MAT原始方法
