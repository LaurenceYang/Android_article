# JAVA并发库concurrent包之ThreadPoolExecutor

标签（空格分隔）： JAVA 异步并发

---

ThreadPoolExecutor是Executor执行框架中最重要的一个实现类，提供了线程池管理和任务管理两个基本功能。

concurrent包提供一系列创建ThreadPoolExecutor的函数，如下：

```java
Executors.newFixedThreadPool(int nThreads); // 创建一个固定线程数的线程池
Executors.newScheduledThreadPool(int nThreads); // 创建一个可对线程进行时间调度的线程池
Executors.newCachedThreadPool(); // 创建一个可缓冲的无线程数量界限(Integer.MAX_VALUE)的线程池
Executors.newSingleThreadExecutor(); // 创建一个可复用的单一线程的线程池
```
再看看创建ThreadPoolExecutor各函数的实现：
```java
//Executors.java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

可以看出，创建不同种类线程池的差距只是在调用ThreadPoolExecutor构造函数的参数不同。

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```
ThreadPoolExecutor参数说明:
> * corePoolSize ： 线程池中核心线程数
> * maximumPoolSize ： 线程池中允许的最大线程数
> * keepAliveTime ： 当前线程数大于核心线程数时，多出的线程在空闲时间超出keepAliveTime时将被终止
> * unit ： keepAliveTime的时间单位
> * workQueue ： 执行前用于保持任务的队列
> * threadFactory ： 执行程序创建新线程使用的工厂
> * handler ： 超出线程范围和队列容量而使执行被阻塞时所使用的处理程序

从参数可以看出，几种线程池的新建主要是“核心线程数”和“最大线程数”的差别，二keepAliveTime和workQueue的差别是根据“核心线程数”和“最大线程数”是否相等决定的。下面看下android23中execute函数的源码：
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```
从代码逻辑可以看到工作线程创建的基本策略：
1.当工作线程数量小于corePoolSize时，通过addWrok(command,true)来创建工作线程处理新建的任务，不入工作队列
2.当工作线程数量大于等于corePoolSize时，先入队列，使用的是BlockingQueue的offer方法。当工作线程数量为0时，还会通过addWorker(null, false)添加一个新的工作线程
3.当工作队列满了并且工作线程数量在corePoolSize和MaximumPoolSize之间，就创建新的工作线程去执行新添加的任务。当工作线程数量超过了MaximumPoolSize，就拒绝任务


##ThreadPoolExecutor主要属性
1、private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));    一个32位的原子整形作为线程池的状态控制描述符。低29位作为工作者线程的数量。所以工作者线程最多有2^29 -1个。高3位来保持线程池的状态。ThreadPoolExecutor总共有5种状态:
  >RUNNING:  可以接受新任务并执行
  SHUTDOWN: 不再接受新任务，但是仍然执行工作队列中的任务
  STOP:     不再接受新任务，不执行工作队列中的任务，并且中断正在执行的任务
  TIDYING:  所有任务被终止，工作线程的数量为0，会去执行terminated()钩子方法
  TERMINATED: terminated()执行结束

下面是一系列ctl这个变量定义和工具方法
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));  
   private static final int COUNT_BITS = Integer.SIZE - 3;  
   private static final int CAPACITY   = (1 << COUNT_BITS) - 1;  
  
   // runState is stored in the high-order bits  
   private static final int RUNNING    = -1 << COUNT_BITS;  
   private static final int SHUTDOWN   =  0 << COUNT_BITS;  
   private static final int STOP       =  1 << COUNT_BITS;  
   private static final int TIDYING    =  2 << COUNT_BITS;  
   private static final int TERMINATED =  3 << COUNT_BITS;  
  
   // Packing and unpacking ctl  
   private static int runStateOf(int c)     { return c & ~CAPACITY; }  
   private static int workerCountOf(int c)  { return c & CAPACITY; }  
   private static int ctlOf(int rs, int wc) { return rs | wc; }  
  
   private static boolean runStateLessThan(int c, int s) {  
       return c < s;  
   }  
  
   private static boolean runStateAtLeast(int c, int s) {  
       return c >= s;  
   }  
  
   private static boolean isRunning(int c) {  
       return c < SHUTDOWN;  
   }  
  
   /**
     * Attempts to CAS-increment the workerCount field of ctl.
     */
   private boolean compareAndIncrementWorkerCount(int expect) {  
       return ctl.compareAndSet(expect, expect + 1);  
   }  
  
  /**
     * Attempts to CAS-decrement the workerCount field of ctl.
     */
   private boolean compareAndDecrementWorkerCount(int expect) {  
       return ctl.compareAndSet(expect, expect - 1);  
   }  
  
  /**
     * Decrements the workerCount field of ctl. This is called only on
     * abrupt termination of a thread (see processWorkerExit). Other
     * decrements are performed within getTask.
     */
   private void decrementWorkerCount() {  
       do {} while (! compareAndDecrementWorkerCount(ctl.get()));  
   }  
```

##工作队列策略
工作队列BlockingQueue<Runnable> workQueue 是用来存放提交的任务的。它有4个基本的策略，并且根据不同的阻塞队列的实现类可以引入更多的工作队列的策略。

4个基本策略：

1. 当工作线程数量小于corePoolSize时，新提交的任务总是会由新创建的工作线程执行，不入队列

2. 当工作线程数量大于corePoolSize，如果工作队列没满，新提交的任务就入队列

3. 当工作线程数量大于corePoolSize，小于MaximumPoolSize时，如果工作队列满了，新提交的任务就交给新创建的工作线程，不入队列

4. 当工作线程数量大于MaximumPoolSize，并且工作队列满了，那么新提交的任务会被拒绝执行。具体看采用何种拒绝策略

根据不同的阻塞队列的实现类，又有几种额外的策略

1. 采用SynchronousQueue直接将任务传递给空闲的线程执行，不额外存储任务。这种方式需要无限制的MaximumPoolSize，可以创建无限制的工作线程来处理提交的任务。这种方式的好处是任务可以很快被执行，适用于任务到达时间大于任务处理时间的情况。**缺点是当任务量很大时，会占用大量线程**

2. 采用无边界的工作队列LinkedBlockingQueue。这种情况下，由于工作队列永远不会满，那么工作线程的数量最大就是corePoolSize，因为当工作线程数量达到corePoolSize时，只有工作队列满的时候才会创建新的工作线程。这种方式好处是使用的线程数量是稳定的，当内存足够大时，可以处理足够多的请求**。缺点是如果任务之间有依赖，很有可能形成死锁，因为当工作线程被消耗完时，不会创建新的工作现场，只会把任务加入工作队列。并且可能由于内存耗尽引发内存溢出OOM**

3. 采用有界的工作队列AraayBlockingQueue。这种情况下对于内存资源是可控的，但是需要合理调节MaximumPoolSize和工作队列的长度，这两个值是相互影响的。当工作队列长度比较小的时，必定会创建更多的线程。而更多的线程会引起上下文切换等额外的消耗。**当工作队列大，MaximumPoolSize小的时候，会影响吞吐量，并且会触发拒绝机制**


参考：
http://www.cnblogs.com/hanmou/p/4622950.html
http://blog.csdn.net/ITer_ZC/article/details/46913841