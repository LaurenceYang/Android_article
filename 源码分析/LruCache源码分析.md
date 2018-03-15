# LruCache源码分析

LruCache要关注的点：

* 内部数据结构
* Cache最大size
* 超过Size后的处理
* 命中率的统计

## 内部数据结构
LruCache是基于LinkedHashmap实现的最近最少使用缓存的类。LinkedHashMap已经实现了最新最老数据的判定，所以使用LinkedHashMap来实现LruCache就很简单。
```java
private final LinkedHashMap<K, V> map;
```

## LruCache最大size
```java
private int maxSize;//由maxSize成员变量保持
```

## get/put
在调用LruCache的get方法后，会首先从LinkedHashmap对象中去取对应key值的value，如果有直接返回。如果没有，通过create创建一个新value，加入到map中。
如果map的size大于最大size，执行超过size后的操作trimToSize。
```java
public final V get(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }

    V mapValue;
    synchronized (this) {
        mapValue = map.get(key);
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        missCount++;
    }

    /*
     * Attempt to create a value. This may take a long time, and the map
     * may be different when create() returns. If a conflicting value was
     * added to the map while create() was working, we leave that value in
     * the map and release the created value.
     */

    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }

    synchronized (this) {
        createCount++;
        mapValue = map.put(key, createdValue);

        if (mapValue != null) {
            // There was a conflict so undo that last put
            map.put(key, mapValue);
        } else {
            size += safeSizeOf(key, createdValue);
        }
    }

    if (mapValue != null) {
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        trimToSize(maxSize);
        return createdValue;
    }
}
```
put的操作比较简单，就是在map中加入新的键值对即可，最后检测trimToSize操作。
```java
public final V put(K key, V value) {
    if (key == null || value == null) {
        throw new NullPointerException("key == null || value == null");
    }

    V previous;
    synchronized (this) {
        putCount++;
        size += safeSizeOf(key, value);
        previous = map.put(key, value);
        if (previous != null) {
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        entryRemoved(false, key, previous, value);
    }

    trimToSize(maxSize);
    return previous;
}
```


## 超过Size后的处理
trimToSize函数会处理超过max size的情况。如果超过max size,会遍历map，删除map的最老数据，然后再重复判断，删除操作，直到cache的size小于max size。
```java
public void trimToSize(int maxSize) {
    while (true) {
        K key;
        V value;
        synchronized (this) {
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(getClass().getName()
                        + ".sizeOf() is reporting inconsistent results!");
            }

            if (size <= maxSize) {
                break;
            }

            Map.Entry<K, V> toEvict = map.eldest();
            if (toEvict == null) {
                break;
            }

            key = toEvict.getKey();
            value = toEvict.getValue();
            map.remove(key);
            size -= safeSizeOf(key, value);
            evictionCount++;
        }

        entryRemoved(true, key, value, null);
    }
}
```

## 命中率的统计
如何评论一个cache设计的好坏，可以用命中率来评估。在LruCache中也加入了这样的统计。
```java
private int putCount;
private int createCount;
private int evictionCount;
private int hitCount;
private int missCount;

@Override public synchronized final String toString() {
    int accesses = hitCount + missCount;
    int hitPercent = accesses != 0 ? (100 * hitCount / accesses) : 0;
    return String.format("LruCache[maxSize=%d,hits=%d,misses=%d,hitRate=%d%%]",
            maxSize, hitCount, missCount, hitPercent);
}
```

从上面的代码可以看出来，只要理解了HashMap及LinkedHashMap，LruCache还是很好理解的。