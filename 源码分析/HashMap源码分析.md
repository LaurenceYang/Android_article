# HashMap源码分析

HashMap要关注的点：

* 内部数据结构
* Hash算法
* 冲突解决方法
* 扩容策略


## 内部数据结构
```java
/**
 * The table, resized as necessary. Length MUST Always be a power of two.
 */
transient HashMapEntry<K,V>[] table = (HashMapEntry<K,V>[]) EMPTY_TABLE;

static class HashMapEntry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    HashMapEntry<K,V> next;
    int hash;

    /**
     * Creates new entry.
     */
    HashMapEntry(int h, K k, V v, HashMapEntry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }

    public final K getKey() {
        return key;
    }

    public final V getValue() {
        return value;
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry e = (Map.Entry)o;
        Object k1 = getKey();
        Object k2 = e.getKey();
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
            Object v1 = getValue();
            Object v2 = e.getValue();
            if (v1 == v2 || (v1 != null && v1.equals(v2)))
                return true;
        }
        return false;
    }

    public final int hashCode() {
        return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());
    }

    public final String toString() {
        return getKey() + "=" + getValue();
    }

    /**
     * This method is invoked whenever the value in an entry is
     * overwritten by an invocation of put(k,v) for a key k that's already
     * in the HashMap.
     */
    void recordAccess(HashMap<K,V> m) {
    }

    /**
     * This method is invoked whenever the entry is
     * removed from the table.
     */
    void recordRemoval(HashMap<K,V> m) {
    }
}
```
table是一个HashMapEntry的数组，HashMapEntry实际为一个链表，每个节点包括key，value和hash。

从这里可以看出Android里面HashMap的实现实际是使用了数组+链表法。

## Hash算法
```java
/**
 * Returns the entry associated with the specified key in the
 * HashMap.  Returns null if the HashMap contains no mapping
 * for the key.
 */
final Entry<K,V> getEntry(Object key) {
    if (size == 0) {
        return null;
    }

    int hash = (key == null) ? 0 : sun.misc.Hashing.singleWordWangJenkinsHash(key);
    for (HashMapEntry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

以上是getEntry的代码，从代码可以看出Android HashMap中使用的hash算法是sun.misc.Hashing.singleWordWangJenkinsHash算法。
Wang／JenkinsHash算法具体内容这里我们不深究。具体可以百度。

## 冲突解决方法
```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    if (key == null)
        return putForNullKey(value);
    int hash = sun.misc.Hashing.singleWordWangJenkinsHash(key);
    int i = indexFor(hash, table.length);
    for (HashMapEntry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);
    return null;
}

void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? sun.misc.Hashing.singleWordWangJenkinsHash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
    HashMapEntry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new HashMapEntry<>(hash, key, value, e);
    size++;
}
```

冲突解决也是利用数组+链表方法解决的；

上面的put函数中，首先计算key的hash，根据hash算对应的索引，
在索引对应的链表里面进行遍历，发现有相同key的数据，进行value的替换；
如果没有，在该链表的最头部加上新节点。

## 扩容策略
```java
void resize(int newCapacity) {
    HashMapEntry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    HashMapEntry[] newTable = new HashMapEntry[newCapacity];
    transfer(newTable);
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

void transfer(HashMapEntry[] newTable) {
    int newCapacity = newTable.length;
    for (HashMapEntry<K,V> e : table) {
        while(null != e) {
            HashMapEntry<K,V> next = e.next;
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

HashMap的默认加载因子是0.75，即达到当前容量的0.75倍时，开始扩容操作，按照double size的容量进行扩容。

扩容后，将老table的数据重新计算每个项的index然后移到新table里面。