# ThreadLocal内存泄漏分析

```
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * <tt>get</tt> or <tt>set</tt> method) has its own, independently initialized
 * copy of the variable.  <tt>ThreadLocal</tt> instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
```

ThreadLocal提供线程本地变量，讲讲为什么会发生内存泄漏？

引用网上的一张图：

![ThreadLocal](http://incdn1.b0.upaiyun.com/2016/10/2fdbd552107780c5ae5f98126b38d5a4.png)


每个Thread对象中持有一个ThreadLocal.ThrealLocalMap对象，Map对象的Key为ThreadLocal实例，
并且为弱引用，不过该弱引用只针对key，不针对value。所以当将ThreadLocal对象设为空，
并且没有强引用指向ThreadLocal对象时，该对象就会被gc回收。
但是Map里面的value不会被回收，因为存在一条从thread过来的强引用，
只有当线程结束后，current thread不存在栈里面，强引用断开，Thread，Map，Value才会被GC回收。

所以得出一个结论就是只要这个线程对象被gc回收，就不会出现内存泄露，
但在threadLocal设为null和线程结束这段时间不会被回收的，就发生了我们认为的内存泄露。
其实这是一个对概念理解的不一致，也没什么好争论的。

最要命的是线程对象不被回收的情况，这就发生了真正意义上的内存泄露。
比如使用线程池的时候，线程结束是不会销毁的，会再次使用的。就可能出现内存泄露。

Java为了最小化减少内存泄露的可能性和影响，
在ThreadLocal的get,set的时候都会清除线程Map里所有key为null的value。
所以最怕的情况就是，threadLocal对象设null了，开始发生“内存泄露”，
然后使用线程池，这个线程结束，线程放回线程池中不销毁，这个线程一直不被使用，
或者分配使用了又不再调用get,set方法，那么这个期间就会发生真正的内存泄露。

解决方法：
使用完ThreadLocal，调用remove，清理数据。