# 换个角度看Android异步通信原理



换个角度理解Android中异步通信问题：



## 首先我们要理解同一进程的两个线程共享哪些资源？

进程代码段</br>

进程的公有数据(**利用这些共享的数据，线程很容易的实现相互之间的通讯**)</br>

进程打开的文件描述符</br>

信号的处理器</br>

进程的当前目录和进程用户ID与进程组ID。</br>



## 其次我们要理解同一进程的两个线程有哪些不同？

线程ID</br>

寄存器组的值</br>

线程的栈(栈是保证线程独立运行所必须的)</br>

错误返回码</br>

线程的信号屏蔽码</br>

线程的优先级</br>



## 所以？

线程间要实现数据的通讯最方便的方式就是利用进程的公有数据。



## 回头来看Handler，Message，Looper关系

每个线程一个Looper实例，同时一个Looper只有一个MessageQueue。</br>

loop()方法，不断从MessageQueue中去取消息，交给消息的target属性的dispatchMessage去处理。</br>

Handler在初始化的时候就会关联上Looper的MessageQueue对象。</br>

</br>

在另外一个线程中，因为可以共享使用Handler对象，就可以通过Handler发送消息给主线程</br>



## 简言之

Handler在主线程创建时会与主线程绑定，同时关联主线程的Looper以及MessageQueue。在子线程时可通过引用该Handler对象发送Message请求将其加入MessageQueue。而主线程中looper会轮询MessageQueue，当有新的Message进入时，会交由handler的handleMessage函数在主线程进行处理。</br>

**核心的原理就是利用了两个线程都使用的公有数据MessageQueue进行了线程间通信。**</br>

是不是很简单～～



