# 简单使用

```java
 Hashtable<String, String> table=new Hashtable<>();
 table.put("T1", "1");
```

Hashtable 的使用方式跟 HasMap 十分相似。

```java
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable {
    //...
}
```

Hashtable 继承了 Dictionary 并实现了 Map，Cloneable，Serializable 接口。

# 节点 HashtableEntry

```java
private static class HashtableEntry<K,V> implements Map.Entry<K,V> {
    // END Android-changed: Renamed Entry -> HashtableEntry.
    final int hash;
    final K key;
    V value;
    HashtableEntry<K,V> next;

    protected HashtableEntry(int hash, K key, V value, HashtableEntry<K,V> next) {
        this.hash = hash;
        this.key =  key;
        this.value = value;
        this.next = next;
    }

    @SuppressWarnings("unchecked")
    protected Object clone() {
        return new HashtableEntry<>(hash, key, value,
                                    (next==null ? null : (HashtableEntry<K,V>) next.clone()));
    }

    // Map.Entry Ops

    public K getKey() {
        return key;
    }

    public V getValue() {
        return value;
    }

    public V setValue(V value) {
        if (value == null)
            throw new NullPointerException();

        V oldValue = this.value;
        this.value = value;
        return oldValue;
    }

    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;

        return (key==null ? e.getKey()==null : key.equals(e.getKey())) &&
            (value==null ? e.getValue()==null : value.equals(e.getValue()));
    }

    public int hashCode() {
        return hash ^ Objects.hashCode(value);
    }

    public String toString() {
        return key.toString()+"="+value.toString();
    }
}
```

Hashtable 将每一个节点封装成了 HashtableEntry。里面包含了 hash 值，key，value 以及下一个节点 next。可见 HashtableEntry 跟 HashMap 一样也是一个单链表结构。
**setValue 方法会判断 value 是否为空，如果空则抛出空指针异常。所以 Hashtable 的 value 值不能为 null。**
**hashCode 方法的返回是 hash 值与 value 的 hashCode 的 亦或 操作返回的，这也是跟 HashMap 的一个区别。**
**
# 构造函数

```java
public Hashtable(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load: "+loadFactor);

    if (initialCapacity==0)
        initialCapacity = 1;
    this.loadFactor = loadFactor;
    table = new HashtableEntry<?,?>[initialCapacity];
    // Android-changed: Ignore loadFactor when calculating threshold from initialCapacity
    // threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    threshold = (int)Math.min(initialCapacity, MAX_ARRAY_SIZE + 1);
}

public Hashtable(int initialCapacity) {
    this(initialCapacity, 0.75f);
}

public Hashtable() {
    this(11, 0.75f);
}

public Hashtable(Map<? extends K, ? extends V> t) {
    this(Math.max(2*t.size(), 11), 0.75f);
    putAll(t);
}
```

逐个往下看，在第一个构造函数中传入的是哈希桶初始化容量 initialCapacity 和 加载因子。

1. initialCapacity 不能小于 0，如果等于 0，则赋值为 1，同时，它没有最大值限制，跟 HashMap 不一样。
2. loadFactor 不能小于等于 0。
3. table 是哈希桶 ，类型是 HashtableEntry<?,?>[] 数组。
4. 阀值 threshold 的计算方法是 初始化容量与 MAX_ARRAY_SIZE+1 直接取最小值，这也是跟 HashMap 不一样的地方。 **MAX_ARRAY_SIZE** 的值是 **Integer.MAX_VALUE - 8。即 2 的 31 次方减 8。**为什么减 8？注释上写了是为力防止请求数组超过VM限制导致OOM。

其他构造函数则是指定了哈希桶容量和加载因子大小。可以看到，**哈希桶容量默认是 11**，这与 HashMap 不一样。
加载因子默认值是 0.75。

最后一个构造函数的作用是新建一个 hash 表。把一个 Hash 表添加到当前表中。它调用了指定哈希桶初始化容量的构造方法 **Hashtable(int initialCapacity)。**initialCapacity 的取值为传进来的 Map 长度的 2 倍和默认值 11 之间取最大值。然后调用 putAll 方法。

# putAll

```java
public synchronized void putAll(Map<? extends K, ? extends V> t) {
    for (Map.Entry<? extends K, ? extends V> e : t.entrySet())
        put(e.getKey(), e.getValue());
}
```

 putAll 方法是遍历 Map 调用 put 方法逐个添加。可以看到，putAll 方法有加锁 synchronized，是线程安全的。

# put

```java
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    //保障 value 方法不能为 null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    HashtableEntry<?,?> tab[] = table;
    //如果 key 为空的话，这里会报异常，所以key不能为 null
    int hash = key.hashCode();
    //得到下标值，hash & 0x7FFFFFFF 是为了避免负数。
    //为什么不用abs（绝对值）？涉及一个int负数最大值问题
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    //根据下标在哈希表中获取节点 entry
    HashtableEntry<K,V> entry = (HashtableEntry<K,V>)tab[index];
    //循环遍历，如果entry不为空，并找到了entry节点，则覆盖 value,并返回旧的 value
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }
	//如果找不到就调用 addEntry 添加
    addEntry(hash, key, value, index);
    return null;
}
```

1.  在 put 方法中会判断 value 是否为 null，null 则抛异常
2. 下标的计算为  **(hash & 0x7FFFFFFF) % tab.length**，hash 值是通过 key 的 hasCode 方法获取的，如果 key 为 null 的话，这里会抛异常，所以** key 和 value 都不能为 null。hash & 0x7FFFFFFF 是为了避免负数。为什么不能用 abs绝对值，因为这里涉及到 int 类型负数最大值问题。可以百度了解一下。**
3. 根据下标 index 在当前哈希表中获取对应的节点 entry。然后循环遍历链表，如果节点不为空并且找到了，那么就会覆盖新的 value 到旧的 value 上，然后返回旧的 value。找不到则调用 addEntry 方法添加。

    ** 所以，Hashtable 相同 value 也是会覆盖的。**
**
# addEntry

```java
private void addEntry(int hash, K key, V value, int index) {
    modCount++; //修改modCount

    HashtableEntry<?,?> tab[] = table;
    if (count >= threshold) { //如果当前哈希桶数量大于阀值
        // Rehash the table if the threshold is exceeded
        //则扩容
        rehash();
		//扩容后重新赋值
        tab = table;
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    //新建节点并添加到当前哈希表中
    HashtableEntry<K,V> e = (HashtableEntry<K,V>) tab[index];
    tab[index] = new HashtableEntry<>(hash, key, value, e);
    count++; //改变 count
}
```

1. 先修改 modCount 值
2. 判断当前哈希桶数量是否大于等于阀值，大于等于的话就要进行扩容。扩容后重新赋值给变量。
3. 哈希表叠加
4. 改变 count

# rehash

```java
protected void rehash() {
    //获取当前哈希表的长度
    int oldCapacity = table.length;
    //保存一下当前哈希表
    HashtableEntry<?,?>[] oldMap = table;

    // overflow-conscious code
    //计算新的哈希表容量，是旧的两倍+1
    int newCapacity = (oldCapacity << 1) + 1;
    //如果新计算出来的容量比最大值MAX_ARRAY_SIZE（Integer.MAX_VALUE - 8）大
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        //如果就的容量刚好等于最大值，就返回，不继续扩容
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        //那么新的容量大小就是最大值。
        newCapacity = MAX_ARRAY_SIZE;
    }
    //根据新的容量新建一个 hash 表
    HashtableEntry<?,?>[] newMap = new HashtableEntry<?,?>[newCapacity];
	//修改 modCount
    modCount++;
    //重新计算阀值
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    //更新当前哈希表
    table = newMap;
    //遍历旧的哈希表，如果有值就把它移到新的哈希表中
    for (int i = oldCapacity ; i-- > 0 ;) {
        //循环判断是否有值
        for (HashtableEntry<K,V> old = (HashtableEntry<K,V>)oldMap[i] ; old != null ; ) {
            HashtableEntry<K,V> e = old; //当前节点 e
            old = old.next; //当前节点的下一个节点赋值跟 old
			//计算下标
            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (HashtableEntry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```

1. 扩容函数首先计算扩容后的哈希表容量为**旧的两倍+1（(oldCapacity << 1) + 1）。**
2. 如果扩容前的容量刚好等于最大容量值的话，就不需要扩容了。扩容后的容量最大值也是 MAX_ARRAY_SIZE
3. 根据新的容量新建一个哈希表。
4. 修改 modCount
5. 重新计算阀值，计算方法为：Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1)
6. 遍历旧的哈希表，如果有值的话，把值拷贝到新的哈希表中。

# putIfAbsent

```java
public synchronized V putIfAbsent(K key, V value) {
    Objects.requireNonNull(value);

    // Makes sure the key is not already in the hashtable.
    HashtableEntry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    HashtableEntry<K,V> entry = (HashtableEntry<K,V>)tab[index];
    for (; entry != null; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            if (old == null) {
                entry.value = value;
            }
            return old;
        }
    }

    addEntry(hash, key, value, index);
    return null;
}
```

这个方法先遍历当前哈希表，如果找到了，则判断 value 是否为空，如果空的话就直接覆盖新的 value 到旧的 value 上，当然新的 value 不能为空。否则就调用 addEntry 添加一个新的 value 进去。

# get

```java
public synchronized V get(Object key) {
    HashtableEntry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (HashtableEntry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return (V)e.value;
        }
    }
    return null;
}
```

根据 key 去查找 value，实现原理是遍历当前哈希表，判断 hash 值和 key 值是否相等，找到则返回，否则返回 null。

# containsKey

```java
public synchronized boolean containsKey(Object key) {
    HashtableEntry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (HashtableEntry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return true;
        }
    }
    return false;
}
```

containsKey 方法跟 get 方法是一样的，只不过是返回值不一样。

# containsValue

```java
public boolean containsValue(Object value) {
    return contains(value);
}

public synchronized boolean contains(Object value) {
    if (value == null) {
        throw new NullPointerException();
    }

    HashtableEntry<?,?> tab[] = table;
    for (int i = tab.length ; i-- > 0 ;) {
        for (HashtableEntry<?,?> e = tab[i] ; e != null ; e = e.next) {
            if (e.value.equals(value)) {
                return true;
            }
        }
    }
    return false;
}
```

containsValue 方法实际上是调用了 contains 方法，contains 方法的实现也是通过遍历当前哈希表，如果 value 一样则返回 true。

# remove

```java
public synchronized V remove(Object key) {
    HashtableEntry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    //查找待删除节点
    HashtableEntry<K,V> e = (HashtableEntry<K,V>)tab[index];
    //prev 是待删除节点的前一个节点
    for(HashtableEntry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            modCount++;
            //如果节点在中间
            if (prev != null) {
                prev.next = e.next;
            } else { //如果在链表头
                tab[index] = e.next;
            }
            count--;
            V oldValue = e.value;
            e.value = null;
            return oldValue;
        }
    }
    return null;
}
```

如果链表只有一个值，则 prev = null，e.next = null。如果 prev 不为空，则待删除的节点在链表中间。则 prev.next = e.next; 让后一个节点覆盖前一个节点。最后返回待删除的节点 value。



# Hashtable 与 HashMap 的主要区别

1. Hashtable 的 key 和 value 都不能为空，HashMap 可以。
2. 因为 Hashtable 的方法都加了锁 synchronized，所以它是线程安全的
3. Hashtable 的默认容量是 11，HashMap 是 16
4. Hashtable 直接使用 key 的 hashCode 作为 hash 值，HashMap 使用扰动函数 hash() 对key 的 hashCode进行扰动后作为 hash 值 .
5. 在取下标时，Hashtable 是直接用模运算 % （int index = (hash & 0x7FFFFFFF) % tab.length;）HashMap 是 （index = (n - 1) & hash]）。因为 Hashtable 默认容量也不是2的n次方。所以也无法用位运算替代模运算。
6. 扩容时，新容量是原来的2倍+1。int newCapacity = (oldCapacity << 1) + 1;
7. Hashtable 是 Dictionary 的子类同时也实现了 Map 接口，HashMap 是 Map 接口的一个实现类；
