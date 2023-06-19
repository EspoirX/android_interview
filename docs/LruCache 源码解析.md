LruCache 的实现与 LinkedHashMap 有关，所以这里分析一下。
先看看 LruCache 的成员变量：

```java
public class LruCache<K, V> {
    private final LinkedHashMap<K, V> map;

    /** Size of this cache in units. Not necessarily the number of elements. */
    private int size;
    private int maxSize;

    private int putCount;
    private int createCount;
    private int evictionCount;
    private int hitCount;
    private int missCount;
    //...
}
```

成员变量不多，但可以看到 LruCache 的主要实现是依赖于 LinkedHashMap 的。

1. size：不同 key-value 条目下缓存的大小，不一定是 key-value 条目的数量
2. maxSize：缓存大小的最大值
3. putCount：存储的 key-value 条目的个数
4. createCount：创建 key 对应的 value 的次数
5. evictionCount：缓存移除的次数
6. hitCount：缓存命中的次数
7. missCount：缓存未命中的次数

# 构造函数

```java
public LruCache(int maxSize) {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }
    this.maxSize = maxSize;
    this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
}
```

在构造函数中，需要指定缓存最大值大小。  
**注释中的 maxSize 定义**：对于不重写 **sizeOf** 方法的缓存，这是缓存中的最大条目数。 对于所有其他缓存，这是此缓存中条目大小的最大和。  
接下来初始化了 LinkedHashMap，传入初始的哈希桶大小为 0，加载因子 0.75，accessOrder 为 true。对于 accessOrder 为 true，通过 LinkedHashMap 的分析知道，访问的时候，输出的顺序是**按照访问节点的顺序。**
 
**比如：**比如原始 Map 中值的顺序是 ABCD，这时查找 AB，那么 LinkedHashMap 重新排序顺序为 CDAB，最后查找的排在 list 尾部，这样等容量超过 LruCache 初始化的最大值 maxSize 时，就可以从 list 头开始删除。

# Size 操作

## safeSizeOf

LruCache 对 size 的操作有好几个方法，这里先看看 safeSizeOf 方法：

```java
private int safeSizeOf(K key, V value) {
    int result = sizeOf(key, value);
    if (result < 0) {
        throw new IllegalStateException("Negative size: " + key + "=" + value);
    }
    return result;
}

protected int sizeOf(K key, V value) {
    return 1;
}
```

safeSizeOf 里面调用了 sizeOf 方法，sizeOf 我们往往需要重写它，默认大小是 1，它的定义是：  
返回 **以用户定义的单位返回 key-value 条目的大小，默认实现返回 1，因此 size 是条目数，max size 是最大条目数。条目在缓存中时，其大小不得更改。**

默认情况下，缓存大小以条目数衡量。 覆写 sizeOf 方法以不同的单位调整缓存大小。 例如，此缓存限于位图的4MiB：

```java
int cacheSize = 4 * 1024 * 1024; // 4MiB
LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(cacheSize) {
    protected int sizeOf(String key, Bitmap value) {
        return value.getByteCount();
    }
}}
```

## trimToSize 

```java
private void trimToSize(int maxSize) {
    while (true) {
        K key;
        V value;
        synchronized (this) {
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(getClass().getName()
                                                + ".sizeOf() is reporting inconsistent results!");
            }
            //哈希表中条目的大小小于指定的大小即终止
            if (size <= maxSize) {
                break;
            }

            // BEGIN LAYOUTLIB CHANGE
            // get the last item in the linked list.
            // This is not efficient, the goal here is to minimize the changes
            // compared to the platform version.
            //获取最后一条记录
            Map.Entry<K, V> toEvict = null;
            for (Map.Entry<K, V> entry : map.entrySet()) {
                toEvict = entry;
            }
            // END LAYOUTLIB CHANGE

            if (toEvict == null) {
                break;
            }
			//移除
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

这个方法的作用是 **删除最旧的条目，直到剩余条目总数小于等于指定的大小。**

1. trimToSize 里面是一个无限循环，首先判断 **size <= maxSize，如果哈希表中的条目大小小于指定大小，则结束。**
2. 通过 for 循环遍历 LinkedHashMap，得到了最后一条记录 toEvict，也是最旧的一条。
3. 如果 toEvict 为空，则返回，否则把它从 LinkedHashMap 中删除，并且修改 size，修改 evictionCount。
4. 调用 entryRemoved 方法。此处 evicted 参数为 true，表明是为了腾出空间而进行的删除条目操作，entryRemoved 默认是空实现，需要自己重写。

## resize

```java
public void resize(int maxSize) {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }

    synchronized (this) {
        this.maxSize = maxSize;
    }
    trimToSize(maxSize);
}
```

resize 方法为调整 maxSize 大小。

# 查询 get

```java
public final V get(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }
	//从 map 中取出对应的 value，不为空，修改命中数加一，返回。否则未命中数加一
    V mapValue;
    synchronized (this) {
        mapValue = map.get(key);
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        missCount++;
    }
	//如果没命中，根据 key 创建一个 value,create 方法需要自己重写。
    V createdValue = create(key);
    //没重写或者返回 null（create 默认返回 null），则返回null
    if (createdValue == null) {
        return null;
    }
	//否则，createCount加一，把创建的 value 放进 map 中
    synchronized (this) {
        createCount++;
        mapValue = map.put(key, createdValue);
		//因为 create 的过程可能比较耗时，当 create 返回时，哈希表可能变得不同。
        if (mapValue != null) {
            // mapValue 不为 null，说明存在一个冲突值，保留之前的 value 值
            map.put(key, mapValue);
        } else {
            //否则修改 size
            size += safeSizeOf(key, createdValue);
        }
    }
	//创建的 createdValue 可以通过 entryRemoved 去释放，这时候 evicted 参数为 false
    if (mapValue != null) {
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        trimToSize(maxSize);
        return createdValue;
    }
}

protected V create(K key) {
    return null;
}
```

1. 判断 key，key 不可为 null。
2. 根据 key 在 map 中取值，如果取得到，则修改 hitCount 变量并返回，否则就是未命中，missCount 加一，这个过程是同步的。
3. 如果没命中，则调用 create 方法创建一个 value，create 默认返回 null，可以重写。这个过程可能会比较耗时，当 create 返回时，哈希表可能变得不同。
4. 然后再判断一下调用 create 创建后的 value 是否为空，空的话直接返回 null，并结束。
5. 如果不为空，则把它存在 map 中。并且修改 createCount。
6. map 的 put 方法正常情况下返回 null，如果不为 null，则代表 map 中有相同的 value，需要覆盖，所以当判断不为 null 时，再调用 put，保留之前的 value ，之所以这样，原因是因为第三点。
7. create 在被调用的时候没有添加额外的同步操作，因此其他线程可能在这个方法执行时访问缓存。最后我们可以通过 entryRemoved 方法释放创建的 createdValue，这时候 evivted 参数为 false，这种情况主要发生在当多个线程同时请求相同的 key （导致创建多个值）时，或者当一个线程调用 put 而另一个线程为其创建值时。

# 存储 put

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

1. put 方法，首先 key 和 value 都不能为 null。
2. 然后调用 map 的 put 方法存储信息，同时修改 putCount 和 size。
3. 如果 put 方法返回值不为空，证明 value 有冲突，发生了覆盖操作，则总体大小应该是不变的，所以 size 又减回去，这个过程是同步的。
4. 返回值 previous 不为空的话，可通过 entryRemoved 释放，这时候 evivted 参数为 false，表明不是为了腾出空间而进行的删除操作。

# 删除 remove

```java
public final V remove(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }

    V previous;
    synchronized (this) {
        previous = map.remove(key);
        if (previous != null) {
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        entryRemoved(false, key, previous, null);
    }

    return previous;
}
```

删除方法是调用 map 的 remove 方法。如果删除成功，则 previous 就会有值，previous 为待删除的节点。删除完后修改 size，并且可以通过  entryRemoved 释放。

# 总结

1. LruCache 的最近少使用算法，其实是通过两个大小：size 和 maxSize，每当一个值被访问，它将被移到队尾。当缓存达到指定的数量时，位于队头的值将被移除，并且可能被 GC 回收。具体实现在 trimToSize 方法中。
2. entryRemoved 方法，如果缓存的值包含需要显式释放的资源，那么需要重写这个方法，第一个参数 evivted 代表是否是为了腾出空间而进行的删除操作，这个参数只有在 trimToSize 中才为 true，其他地方都为 false。
3. 在 get 方法中，如果 key 对应的缓存未命中，通过重写 create 方法创建对应的 value。这可以简化代码调用：即使存在缓存未命中，也允许假设始终返回一个值。
4. 默认情况下，缓存大小以条目数量度量。在不同缓存对象下，通过重写 sizeOf 方法测量 key-value 缓存的大小，如上面举例的位图大小。
5. put，get，remove 中，主要的代码块被 synchronized 包裹着，所以这些核心操作是线程安全的。
