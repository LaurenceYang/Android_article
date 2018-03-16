# ConcurrentHashMap源码分析

ConcurrentHashMap中我们要关注的点：

* 红黑树
* 底层存储结构
* 如何解决多线程并发安全问题
* 多线程并发扩容

## 树的基础知识

### 二叉排序树(binary sort tree)
简称BST，左子树小于根节点，右子树大于根节点，左右子树也为二叉排序树。

### AVL tree
根据发明者Adelson-Velsky and Landis的名字命名，是一个自平衡二叉树。
它的左子树和右子树深度之差的绝对值不超过1，且它的左子树和右子树也为AVL树。

AVL树是为了解决BST查询的最坏时间问题，AVL的查询时间总能保证时间复杂度为O(logn)

AVL树的增加和删除操作要照顾平衡，如果平衡因子的绝对值大于1，就要进行调整来维持平衡规则。

调整的方法为旋转，包括左旋转和右旋转。

## 红黑树
因为ConcurrentHashMap中使用到了红黑树，我们先回顾以下红黑树的相关知识。
红黑树也是为了解决二叉排序树在插入数据后的不平衡，也是一种自平衡的二叉树。
红黑树的五个规则：
* 节点是红色或者黑色
* 根节点是黑色
* 每个叶节点（NIL或者空节点）为黑色
* 每个红色节点的两个子节点是黑色的
* 从任一节点到其末个叶节点的所有路径包含相同数量的黑色节点

正是这些规则，保证了红黑树的自平衡。红黑树从根到叶子的最长路径不会超过最短路径的2倍。
当插入或者删除节点的时候，红黑树的规则有可能被打破。这时候就需要做出一些调整，来继续维持我们的规则。

调整的方法包括变色和旋转，旋转又包括左旋转和右旋转。

## 红黑树的演变，为什么有红黑的含义
了解红黑树之前，要先了解另一种树，叫2-3树，黑红树背后的逻辑就是它。

2节点即普通节点，包含一个元素，和两个子节点。

3节点包含2个元素，和三个子节点。如元素A和B，左节点小于A，中节点在A和B之间，右节点大于B。

 我们将{7,8,9,10,11,12}中的数值依次插入2-3树，画出它的过程：

 ![2-3树插入过程](http://img.blog.csdn.net/20160913140427290?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 所以，2-3树的设计完全可以保证二叉树保持矮矮胖胖的状态，保持其性能良好。但是，将这种直白的表述写成代码实现起来并不方便，因为要处理的情况太多。这样需要维护两种不同类型的节点，将链接和其他信息从一个节点复制到另一个节点，将节点从一种类型转换为另一种类型等等。

为了更好的利用2-3-4树平衡高度的特点，同时又更好的便于实现，我们就引入了红黑树。

因此，红黑树出现了，红黑树背后的逻辑就是2-3树的逻辑，但是由于用红黑标记这种小技巧，最后实现的代码量并不大。(但是，要直接理解这些代码是如何工作的以及背后的道理，就比较困难了。所以你一定要理解它的演化过程，才能真正的理解红黑树)

更详细的说明可以参考

1、[清晰理解红黑树的演变---红黑的含义](http://blog.csdn.net/chen_zhang_yu/article/details/52415077)

2、[一篇文章搞懂红黑树的原理及实现](https://www.jianshu.com/p/37c845a5add6)

3、[Red Black.pdf](http://www.cs.princeton.edu/~rs/talks/LLRB/RedBlack.pdf)


## ConcurrentHashMap底层存储结构
ConcurrentHashMap在jdk1.7的结构是使用segments+tables数组+HashEntry单向链表的结构。采用的锁是基于segment的，粒度较大。

在jdk1.8中结构改成了tables数据+单项链表+红黑树的结构。采用的锁是基于tables元素，粒度比1.7的小很多。

## 如何解决多线程并发安全问题
jdk1.7采用Segment作为线程安全锁。Segment本身是一个ReentrantLock。
```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
        transient volatile HashEntry<K,V>[] table;
        transient int count;
    }
```

jdk1.8中使用synchronized关键字对每个链表的首元素进行加锁操作。
```java
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

## 多线程并发扩容
ConcurrentHashMap在多线程时，如果同时有多个线程在操作，其中一个操作触发扩容操作后，会让其它线程一起帮助扩容，
而不是只有检查到要扩容的那个线程进行扩容操作，其他线程就要等待扩容操作完成才能工作。

扩容过程有点复杂，这里主要涉及到多线程并发扩容,ForwardingNode的作用就是支持扩容操作，
将已处理的节点和空节点置为ForwardingNode，并发处理时多个线程经过ForwardingNode就表示已经遍历了，就往后遍历。