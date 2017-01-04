#Android AsyncTask源码分析

标签（空格分隔）： Android 源码 异步并发

---

Android系统自带的AsyncTask一直以来都被诟病。3.0之前当任务队列超过128时，系统会抛出RejectedExecutionException导致系统崩溃。在3.0之后，Android团队又将AsyncTask改成了串行，并可通过设置executeOnExecutor(Executor)来实现多个AsyncTask并行。下面从源码来看下AsyncTask的原理,源码基于android23。

##串行执行器
AsyncTask中定义了一个默认的执行器SERIAL_EXECUTOR，它是一个静态变量，所有的AsyncTask共用该执行器。如果你不通过executeOnExecutor设定其它的执行器，该执行器会控制任务的调用流程，也就是默认的串行调用。SerialExecutor中有一个双向列表mTasks用于存放Runnable，在执行时会将Runable放在mTasks列表中，在前一个runnable运行完成后，会调用scheduleNext从mTasks头部取出Runnable执行。能看出默认的执行器不是真正任务并发处理，而是顺序处理各个任务。
```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

##ThreadPoolExecutor
Java的异步处理基本都是使用concurrent包，这要感谢我们的Doug Lea大神。我们看下默认串行执行器中使用的线程池时什么样子的，需要重点关注的是在创建ThreadPoolExecutor是传入的参数：
> * CORE_POOL_SIZE为CPU数+1，这样可以控制同时并发的数量，合理利用CPU
> * MAXIMUM_POOL_SIZE为CPU_COUNT * 2 + 1
> * KEEP_ALIVE时间为1S
> * 工作队列为长度为128的无边界队列

如果单看此处的线程池运行器的定义，它定义了一个核心线程数为CPU_COUNT + 1，最大线程数为CPU_COUNT * 2 + 1，任务队列为128长度的一个线程池。它的最大并发能力为CPU_COUNT * 2 + 1，当任务数超过128+CPU_COUNT * 2 + 1个是会抛出RejectedExecutionException异常。

但其实在此处的THREAD_POOL_EXECUTOR中，每次都只有一个Runnable投入，感觉不是很合理，可能是以前的并行版本留下的痕迹。
```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE = 1;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};

private static final BlockingQueue<Runnable> sPoolWorkQueue =
    new LinkedBlockingQueue<Runnable>(128);

/**
* An {@link Executor} that can be used to execute tasks in parallel.
*/
public static final Executor THREAD_POOL_EXECUTOR
    = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
            TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```

##executeOnExecutor
要使用AsyncTask实现并发，可以通过调用executeOnExecutor来设置自己的Executor。
```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```

concurrent包中自带的执行器即可实现并发，其中通过调用newCachedThreadPool生成的Executor，对于并发数量没有限制，如投入300个任务，300个任务会同时被执行，但是这种情况容易产生的问题会占用大量线程。
通过newFixedThreadPool生成的Executor，当投入300任务时，最多同时并发10个任务。
```java
Executor executor = Executors.newCachedThreadPool();
Executor executor = Executors.newFixedThreadPool(10);
```


参考：
http://blog.csdn.net/boyupeng/article/details/49001215



