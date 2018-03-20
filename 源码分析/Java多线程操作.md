# Java多线程操作

Java多线程中，我们经常做的是线程间的同步操作。

关于同步的方式，关注以下几种：

* 使用wait,notify
* 使用关键字synchronized
* 使用关键字volatile
* 使用Lock及重入锁ReenTrantLock
* 使用ThreadLocal

## 使用wait，notify
wait()、notify/notifyAll() 方法是Object的本地final方法，无法被重写。

wait()导致线程进入等待状态，直到它被其它线程通过notify和notifyAll唤醒。

notifyAll解除所有那些在该对象上调用wait方法的线程的阻塞状态。

经常使用wait和notify解决经典的生产者消费者问题。

现在你知道wait应该永远在被synchronized的背景下和那个被多线程共享的对象上调用，下一个一定要记住的问题就是，你应该永远在while循环，而不是if语句中调用wait。因为线程是在某些条件下等待的——在我们的例子里，即“如果缓冲区队列是满的话，那么生产者线程应该等待”，你可能直觉就会写一个if语句。但if语句存在一些微妙的小问题，导致即使条件没被满足，你的线程你也有可能被错误地唤醒。所以如果你不在线程被唤醒后再次使用while循环检查唤醒条件是否被满足，你的程序就有可能会出错——例如在缓冲区为满的时候生产者继续生成数据，或者缓冲区为空的时候消费者开始小号数据。所以记住，永远在while循环而不是if语句中使用wait！

基于以上认知，下面这个是使用wait和notify函数的规范代码模板：

```java
// The standard idiom for calling the wait method in Java
synchronized (sharedObject) {
    while (condition) {
        sharedObject.wait();
        // (Releases lock, and reacquires on wakeup)
    }
    // do action based upon condition e.g. take or put into queue.
}
```

以下是生产者消费者模式的代码：
```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.Random;
/**
* Simple Java program to demonstrate How to use wait, notify and notifyAll()
* method in Java by solving producer consumer problem.
*
* @author Javin Paul
*/
public class ProducerConsumerInJava {
    public static void main(String args[]) {
        System.out.println("How to use wait and notify method in Java");
        System.out.println("Solving Producer Consumper Problem");
        Queue buffer = new LinkedList();
        int maxSize = 10;
        Thread producer = new Producer(buffer, maxSize, "PRODUCER");
        Thread consumer = new Consumer(buffer, maxSize, "CONSUMER");
        producer.start();
        consumer.start(); }
    }
    /**
    * Producer Thread will keep producing values for Consumer
    * to consumer. It will use wait() method when Queue is full
    * and use notify() method to send notification to Consumer
    * Thread.
    *
    * @author WINDOWS 8
    *
    */
    class Producer extends Thread {
        private Queue queue;
        private int maxSize;
        public Producer(Queue queue, int maxSize, String name) {
            super(name);
            this.queue = queue;
            this.maxSize = maxSize;
        }

        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    while (queue.size() == maxSize) {
                        try {
                            System.out .println("Queue is full, " + "Producer thread waiting for " + "consumer to take something from queue");
                            queue.wait();
                        } catch (Exception ex) {
                            ex.printStackTrace();
                        }
                    }
                    Random random = new Random();
                    int i = random.nextInt();
                    System.out.println("Producing value : " + i);
                    queue.add(i);
                    queue.notifyAll();
                    }
                }
            }
        }
    /**
    * Consumer Thread will consumer values form shared queue.
    * It will also use wait() method to wait if queue is
    * empty. It will also use notify method to send
    * notification to producer thread after consuming values
    * from queue.
    *
    * @author WINDOWS 8
    *
    */
    class Consumer extends Thread {
        private Queue queue;
        private int maxSize;
        public Consumer(Queue&lt queue, int maxSize, String name){
            super(name);
            this.queue = queue;
            this.maxSize = maxSize;
        }
        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    while (queue.isEmpty()) {
                        System.out.println("Queue is empty," + "Consumer thread is waiting" + " for producer thread to put something in queue");
                        try {
                            queue.wait();
                        } catch (Exception ex) {
                            ex.printStackTrace();
                        }
                    }
                    System.out.println("Consuming value : " + queue.remove());
                    queue.notifyAll();
                }
            }
        }
    }
```


## 使用关键字synchronized
synchronized是通过对象内部的一个叫做monitor（监控器锁）来实现的。但是监视器本质又是依赖于底层操作系统的Mutex Lock实现的。
而操作系统实现线程之间的切换这就需要从用户态转换到核心态，这个成本非常高，状态之间的转换需要相对比较长的时间，这就是为什么Synchronized效率低的原因。
这种依赖于操作系统的Mutex Lock的锁，我们称为"重量级锁"。
JDK中对Synchronized做的种种优化，其核心都是为了减少这种重量级锁的使用。
JDK1.6以后，为了减少获得锁和释放锁所带来的性能消耗，提高性能，引入了“轻量级锁”和“偏向锁”。


## 使用关键字volatile
* 为域变量的访问提供一种面锁机制
* 使用volatile修饰域相当于告诉虚拟机该域可能会被其他线程更新
* 因此每次使用该域就要重新计算，而不是使用寄存器中的值
* volatile不会提供任何原子操作，它也不能用来修饰final类型的变量

注意：因为volatile不能保证原子操作导致的，因此volatile不能代替synchronized。
此外volatile会组织编译器对代码优化，因此能不使用它就不适用它吧。
它的原理是每次要线程要访问volatile修饰的变量时都是从内存中读取，
而不是存缓存当中读取，因此每个线程访问到的变量值都是一样的。这样就保证了同步。

## 使用Lock及重入锁ReenTrantLock
如果synchronized关键字能满足用户的需求，就用synchronized，因为它能简化代码 。
如果需要更高级的功能，就用ReentrantLock类，此时要注意及时释放锁，否则会出现死锁，通常在finally代码释放锁。

## 使用ThreadLocal
ThreadLocal与同步机制
a.ThreadLocal与同步机制都是为了解决多线程中相同变量的访问冲突问题
b.前者采用以"空间换时间"的方法，后者采用以"时间换空间"的方式
