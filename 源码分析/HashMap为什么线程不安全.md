# HashMap为什么线程不安全

HashMap在put，resize等操作都没有做线程安全操作，多线程下肯定会有问题，这一部分看代码就很好理解。
