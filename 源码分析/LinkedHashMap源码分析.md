# LinkedHashMap源码分析

LinkedHashMap需要关注的点：

* 内部数据结构
* 和HashMap的不同处
* 扩容的不同处
* LinkedHashMap的利用场景

## 内部数据结构
基本结构还是使用数组+链的方式，但链从单向链改为了双向链表。
```java
/**
 * LinkedHashMap entry.
 */
private static class LinkedHashMapEntry<K,V> extends HashMapEntry<K,V> {
    // These fields comprise the doubly linked list used for iteration.
    LinkedHashMapEntry<K,V> before, after;

    LinkedHashMapEntry(int hash, K key, V value, HashMapEntry<K,V> next) {
        super(hash, key, value, next);
    }

    /**
     * Removes this entry from the linked list.
     */
    private void remove() {
        before.after = after;
        after.before = before;
    }

    /**
     * Inserts this entry before the specified existing entry in the list.
     */
    private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
        after  = existingEntry;
        before = existingEntry.before;
        before.after = this;
        after.before = this;
    }

    /**
     * This method is invoked by the superclass whenever the value
     * of a pre-existing entry is read by Map.get or modified by Map.set.
     * If the enclosing Map is access-ordered, it moves the entry
     * to the end of the list; otherwise, it does nothing.
     */
    void recordAccess(HashMap<K,V> m) {
        LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
        if (lm.accessOrder) {
            lm.modCount++;
            remove();
            addBefore(lm.header);
        }
    }

    void recordRemoval(HashMap<K,V> m) {
        remove();
    }
}
```
## 和HashMap的不同处
LinkedHashMap初始化时，会创建一个header，这个header的类型为LinkedHashMapEntry，
header.before和header.after都指向自己。
```java
/**
 * Called by superclass constructors and pseudoconstructors (clone,
 * readObject) before any entries are inserted into the map.  Initializes
 * the chain.
 */
@Override
void init() {
    header = new LinkedHashMapEntry<>(-1, null, null, null);
    header.before = header.after = header;
}
```
在添加新的Entry后，除了HashMap添加Entry的操作以外，会在header的前面添加该Entry，
这样就形成了一个循环链表，header.before指向最新加入的Entry，header.after指向最早（也可以理解为最老eldest）的Entry。
```java
void createEntry(int hash, K key, V value, int bucketIndex) {
    HashMapEntry<K,V> old = table[bucketIndex];
    LinkedHashMapEntry<K,V> e = new LinkedHashMapEntry<>(hash, key, value, old);
    table[bucketIndex] = e;
    e.addBefore(header); //在header的前面添加该Entry
    size++;
}
```

在我们访问某个Entry的时候，会将该Entry从循环链表中去掉，然后加到header的前面（即header.before指向该节点），
这样该结点就称为我们访问的最新节点。
```java
void recordAccess(HashMap<K,V> m) {
    LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
    if (lm.accessOrder) {
        lm.modCount++;
        remove(); //删除当前节点
        addBefore(lm.header);把当前节点加到header的前面
    }
}
```

## 扩容的不同处
HashMap在扩容后会做一个旧table到新table的迁移，通过重新计算各个Entry对应的索引，从旧table迁移到新table对应的位置。
采用的遍历方式是遍历table和遍历链表两重遍历。

LinkedHashMap在扩容时唯一不同的就是遍历的方式，因为在原来的结构基础上添加了一个循环链表，
所以LinkedHashMap在遍历是就直接使用该循环链表。如下：
```java
@Override
void transfer(HashMapEntry[] newTable) {
    int newCapacity = newTable.length;
    for (LinkedHashMapEntry<K,V> e = header.after; e != header; e = e.after) {
        int index = indexFor(e.hash, newCapacity);
        e.next = newTable[index];
        newTable[index] = e;
    }
}
```

## LinkedHashMap的利用场景
目前流传最广的利用场景的是LruCache，最新被访问的数据在循环链表的末端，最老的数据在循环链表的前段，
这样就可以通过访问的时间先后次序来删除最老数据，缓存最新数据的目的。

但是在LinkedHashMap的代码中，并没有实现删除最老数据的操作，这一部分可以在LruCache中特别关注下。
