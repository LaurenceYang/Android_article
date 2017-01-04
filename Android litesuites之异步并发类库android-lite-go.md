# Android litesuites之异步并发类库android-lite-go

标签（空格分隔）： Android 开源项目 异步并发

---

github地址:[请点击这里](https://github.com/litesuits/android-lite-go)

**先看下作者对LiteGo的描述**：
>LiteGo是一款基于Java语言的「异步并发类库」，它的核心是一枚「迷你」并发器，它可以自由地设置同一时段的最大「并发」数量，等待「排队」线程数量，还可以设置「排队策略」和「超载策略」。 LiteGo可以直接投入Runnable、Callable、FutureTask 等类型的实现来运行一个任务，它的核心组件是「SmartExecutor」，它可以用来作为「App」内支持异步并发的唯一组件。 **在一个App中「SmartExecutor」可以有多个实例，每个实例都有完全的「独立性」，比如独立的「核心并发」、「排队等待」指标，独立的「运行调度和满载处理」策略，但所有实例「共享一个线程池」。 这种机制既满足不同模块对线程控制和任务调度的独立需求，又共享一个池资源来节省开销，最大程度上节约资源复用线程，帮助提升性能。**

**我们需要理解的是，LiteGo解决了我们在使用concurrent库中可能出现的哪些问题。**
我们平常在异步处理中，一般会使用Executors.newCachedThreadPool()去创建一个缓存线程库。当然，也可能使用如newFixedThreadPool去创建固定线程池方法等等。这里以常用的newCacheThreadPool为例讨论该问题。看下newCachedThreadPool的创建代码，参数中corePoolSize为0，maximumPoolSize为MAX_VAULE,workQueue为SynchronousQueue，我们每次在投入任务（如Runnable时）的时候，SynchronousQueue会直接将任务传递给空闲的线程执行，不额外存储任务。这种方式的好处是任务可以很快被执行，适合于任务到达时间大于任务处理时间的情况。缺点是当任务量很大时，会占用大量线程，影响吞吐量。另外，空闲工作线程会在60s后被回收，直到为0，下次再使用线程的时候，又需要重新创建线程，线程的创建代价比较大，其实可以考虑持有少量的核心进程不让其被回收。
```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

个人觉得litego是主要为了解决同时并发过大时的吞吐量问题，一家之言，欢迎抛砖。
**LiteGo理念**
>* 清闲时线程不要多持，最好不要超过CPU数量，根据具体应用类型和场景来决策。
>* 瞬间并发不要过多，最好保持在CPU数量左右，或者可以多几个问题并不大。
>* 注意控制排队和满载策略，大量并发瞬间起来的场景下也能轻松应对。
同时并发的线程数量不要过多，最好保持在CPU核数左右，过多了CPU时间片过多的轮转分配造成吞吐量降低，过少了不能充分利用CPU，并发数可以适当比CPU核数多一点没问题。

**LiteGo实现**
LiteGo在创建缓存池时在Executors.newCachedThreadPool()的基础上进行修改,持有的核心线程数改成了根据CPU数量决定，这样在清闲时也不会多持有，在有任务投放时也可以使用持有的缓存线程。
```java
public static ThreadPoolExecutor createDefaultThreadPool() {
    // 控制最多4个keep在pool中
    int corePoolSize = Math.min(4, CPU_CORE);
    return new ThreadPoolExecutor(
            corePoolSize,
            Integer.MAX_VALUE,
            DEFAULT_CACHE_SENCOND, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>(),
            new ThreadFactory() {
                static final String NAME = "lite-";
                AtomicInteger IDS = new AtomicInteger(1);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, NAME + IDS.getAndIncrement());
                }
            },
            new ThreadPoolExecutor.DiscardPolicy());
}
```


「排队策略」和「超载策略」也在原有Executor策略的基础上做了一套自己的封装。
```java
public void execute(final Runnable command) {
        if (command == null) {
            return;
        }

        WrappedRunnable scheduler = new WrappedRunnable() {
            @Override
            public Runnable getRealRunnable() {
                return command;
            }

            @Override
            public void run() {
                try {
                    command.run();
                } finally {
                    scheduleNext(this);
                }
            }
        };

        boolean callerRun = false;
        synchronized (lock) {
            //if (debug) {
            //    Log.v(TAG, "SmartExecutor core-queue size: " + coreSize + " - " + queueSize
            //                   + "  running-wait task: " + runningList.size() + " - " + waitingList.size());
            //}
            if (runningList.size() < coreSize) {
                runningList.add(scheduler);
                threadPool.execute(scheduler);
                //Log.v(TAG, "SmartExecutor task execute");
            } else if (waitingList.size() < queueSize) {
                waitingList.addLast(scheduler);
                //Log.v(TAG, "SmartExecutor task waiting");
            } else {
                //if (debug) {
                //    Log.w(TAG, "SmartExecutor overload , policy is: " + overloadPolicy);
                //}
                switch (overloadPolicy) {
                    case DiscardNewTaskInQueue:
                        waitingList.pollLast();
                        waitingList.addLast(scheduler);
                        break;
                    case DiscardOldTaskInQueue:
                        waitingList.pollFirst();
                        waitingList.addLast(scheduler);
                        break;
                    case CallerRuns:
                        callerRun = true;
                        break;
                    case DiscardCurrentTask:
                        break;
                    case ThrowExecption:
                        throw new RuntimeException("Task rejected from lite smart executor. " + command.toString());
                    default:
                        break;
                }
            }
            //printThreadPoolInfo();
        }
        if (callerRun) {
            if (debug) {
                Log.i(TAG, "SmartExecutor task running in caller thread");
            }
            command.run();
        }
    }
```

```java
 private void scheduleNext(WrappedRunnable scheduler) {
        synchronized (lock) {
            boolean suc = runningList.remove(scheduler);
            //if (debug) {
            //    Log.v(TAG, "Thread " + Thread.currentThread().getName()
            //                   + " is completed. remove prior: " + suc + ", try schedule next..");
            //}
            if (!suc) {
                runningList.clear();
                Log.e(TAG,
                        "SmartExecutor scheduler remove failed, so clear all(running list) to avoid unpreditable error : " + scheduler);
            }
            if (waitingList.size() > 0) {
                WrappedRunnable waitingRun;
                switch (schedulePolicy) {
                    case LastInFirstRun:
                        waitingRun = waitingList.pollLast();
                        break;
                    case FirstInFistRun:
                        waitingRun = waitingList.pollFirst();
                        break;
                    default:
                        waitingRun = waitingList.pollLast();
                        break;
                }
                if (waitingRun != null) {
                    runningList.add(waitingRun);
                    threadPool.execute(waitingRun);
                    Log.v(TAG, "Thread " + Thread.currentThread().getName() + " execute next task..");
                } else {
                    Log.e(TAG,
                            "SmartExecutor get a NULL task from waiting queue: " + Thread.currentThread().getName());
                }
            } else {
                if (debug) {
                    Log.v(TAG, "SmartExecutor: all tasks is completed. current thread: " +
                               Thread.currentThread().getName());
                    //printThreadPoolInfo();
                }
            }
        }
    }
```

这样就实现了尽可能利用concurrent包，并且控制了同时并发的线程数量，合理的利用了CPU。

>个人觉得该开源项目只能算中规中矩，因为使用了自己封装的排队策略，是否比直接使用Executor有效还有待项目实践！！因为解决的多并发问题不一定是概率较高的一个问题！！