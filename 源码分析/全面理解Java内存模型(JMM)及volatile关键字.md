# 全面理解Java内存模型(JMM)及volatile关键字

参考[全面理解Java内存模型(JMM)及volatile关键字](http://blog.csdn.net/javazejian/article/details/72772461)


## volatile内存语义
* 保证被volatile修饰的共享变量对所有线程总数可见的，也就是当一个线程修改了一个被volatile修饰共享变量的值，新值总数可以被其他线程立即得知。

* 禁止指令重排序优化

## volatile的可见性
那么JMM是如何实现让volatile变量对其他线程立即可见的呢？实际上，当写一个volatile变量时，JMM会把该线程对应的工作内存中的共享变量值刷新到主内存中，当读取一个volatile变量时，JMM会把该线程对应的工作内存置为无效，那么该线程将只能从主内存中重新读取共享变量。volatile变量正是通过这种写-读方式实现对其他线程可见（但其内存语义实现则是通过内存屏障）

## volatile禁止重排优化
volatile关键字另一个作用就是禁止指令重排优化，从而避免多线程环境下程序出现乱序执行的现象。

内存屏障，又称内存栅栏，是一个CPU指令，它的作用有两个，一是保证特定操作的执行顺序，二是保证某些变量的内存可见性（利用该特性实现volatile的内存可见性）。由于编译器和处理器都能执行指令重排优化。如果在指令间插入一条Memory Barrier则会告诉编译器和CPU，不管什么指令都不能和这条Memory Barrier指令重排序，也就是说通过插入内存屏障禁止在内存屏障前后的指令执行重排序优化。Memory Barrier的另外一个作用是强制刷出各种CPU的缓存数据，因此任何CPU上的线程都能读取到这些数据的最新版本。总之，volatile变量正是通过内存屏障实现其在内存中的语义，即可见性和禁止重排优化。

## 说说单例的问题
```java
public class DoubleCheckLock {

    private static DoubleCheckLock instance;

    private DoubleCheckLock(){}

    public static DoubleCheckLock getInstance(){

        //第一次检测
        if (instance==null){   //某一个线程执行到第一次检测，读取到的instance不为null时，instance的引用对象可能没有完成初始化。
            //同步
            synchronized (DoubleCheckLock.class){
                if (instance == null){
                    //多线程环境下可能会出现问题的地方
                    instance = new DoubleCheckLock(); //1
                }
            }
        }
        return instance;
    }
}
```
 到这里已经很完美了，看起来没有问题。
 但是这种双重检测机制在JDK1.5之前是有问题的，
 问题还是出在//1，由所谓的无序写入造成的。
 一般来讲，当初始化一个对象的时候，会经历内存分配、初始化、返回对象在堆上的引用等一系列操作，
 这种方式产生的对象是一个完整的对象，可以正常使用。
 但是JAVA的无序写入可能会造成顺序的颠倒，即内存分配、返回对象引用、初始化的顺序，
 这种情况下对应到//1就是singleton已经不是null，而是指向了堆上的一个对象，但是该对象却还没有完成初始化动作。
 当后续的线程发现singleton不是null而直接使用的时候，就会出现意料之外的问题。


那么该如何解决呢，很简单，我们使用volatile禁止instance变量被执行指令重排优化即可。
```java
//禁止指令重排优化
private volatile static DoubleCheckLock instance;
```
