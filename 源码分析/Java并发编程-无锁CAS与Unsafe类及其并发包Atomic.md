# Java并发编程-无锁CAS与Unsafe类及其并发包Atomic

参考[Java并发编程-无锁CAS与Unsafe类及其并发包Atomic](http://blog.csdn.net/javazejian/article/details/72772470)

说起无锁，先说下乐观派和悲观派，对于乐观派来说，发生坏的事情概率很低，
对于悲观派，发生坏的事情概率很高。无锁和加锁即是对应乐观派和悲观派两种策略。

在java中，无锁策略提供CAS的技术来保证线程间的安全。

CAS，全称是compare and swap，其核心算法是：
```
执行函数：CAS(V,E,N)
```
* V表示要更新的变量
* E表示预期值
* N表示新值

如果V值等于E值，则将V设为N。如果V值不等于E值，代表有其它线程对V做了操作，
此时线程什么也不做。

CAS是如何保证线程安全的呢？CAS是一种系统原语，属于操作系统用于范围，
是由若干条指令组成的，在执行过程中不能被中断。

也就是说，CAS的操作是具有原子性的。

Unsafe类是在sun.misc包下，先了解下即可。

然后是并发包的Atomic，如AtomicInteger，是原子更新整型。
其内部通过Unsafe类的相关操作实现的，这也同时证明了AtomicInteger是基于无锁实现的。

再说说自旋锁，自旋锁假设不久的将来，当前线程能够获得锁，然后就让当前线程做空循环，
在经历过若干个空循环后，如果得到锁，就进入临界区。

我可以通过AtomicReference实现简单的自旋锁
```java
public class SpinLock {
  private AtomicReference<Thread> sign =new AtomicReference<>();

  public void lock(){
    Thread current = Thread.currentThread();
    while(!sign .compareAndSet(null, current)){
    }
  }

  public void unlock (){
    Thread current = Thread.currentThread();
    sign .compareAndSet(current, null);
  }
}
```

事实上，AtomicInteger内部的CAS操作实际也是在while循环中实现的，
不同的是，它结束的条件是线程成功对应的值，也是自旋的一种。