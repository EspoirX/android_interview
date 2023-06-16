**分析之前，先说几个知识点：**

1. ArrayMap 是 android 包下的，并不是 java 包下的。
2. << 的意思是2次幂倍大，>> 的意思是2的2次幂小，举个例子：1<<1 = 2，1<<2 = 4，2>>1 = 1，8>>2 = 2，结果只有整数。
3. ～号的意思是取负数减一，例子：～1 = -2，～2 = -3，～0 = -1，～-5 = 4

ArrayMap 实现了 Map 接口：

```java
public final class ArrayMap<K, V> implements Map<K, V> {
	//...
}
```

# 构造函数

```java
public ArrayMap() {
    this(0, false);
}
 
public ArrayMap(int capacity) {
    this(capacity, false);
}

/** {@hide} */
public ArrayMap(int capacity, boolean identityHashCode) {
    mIdentityHashCode = identityHashCode;
    if (capacity < 0) {
        mHashes = EMPTY_IMMUTABLE_INTS;
        mArray = EmptyArray.OBJECT;
    } else if (capacity == 0) {
        mHashes = EmptyArray.INT;
        mArray = EmptyArray.OBJECT;
    } else {
        allocArrays(capacity);
    }
    mSize = 0;
}

public ArrayMap(ArrayMap<K, V> map) {
    this();
    if (map != null) {
        putAll(map);
    }
}
```

可以看到 ArrayMap 的构造函数有四个，**capacity** 参数是初始化容量，**identityHashCode** 决定了是否使用 System.identityHashCode 方法去获取 hashCode。我们可以先看看它的成员变量定义：


```java
public final class ArrayMap<K, V> implements Map<K, V> {
    //决定发生并发错误时是否抛出 ConcurrentModificationException 异常，默认抛出
    private static final boolean CONCURRENT_MODIFICATION_EXCEPTIONS = true;
    //扩容时，容量增量的最小值
    private static final int BASE_SIZE = 4;
    //缓存数组的上限
    private static final int CACHE_SIZE = 10;
    //空数组，final 修饰代表是不可变的
    static final int[] EMPTY_IMMUTABLE_INTS = new int[0];
    public static final ArrayMap EMPTY = new ArrayMap<>(-1);
    //用于缓存大小为 4 的 ArrayMap
    static Object[] mBaseCache;
    //记录着当前已缓存的数量
    static int mBaseCacheSize;
    //用于缓存大小为 8 的 ArrayMap
    static Object[] mTwiceBaseCache;
    //记录着当前已缓存的数量
    static int mTwiceBaseCacheSize;
	//是否使用 System.identityHashCode 方法去获取 hashCode
    final boolean mIdentityHashCode;
    //由 key 的 hashcode 所组成的数组
    int[] mHashes;
    //由 key-value 对所组成的数组，是 mHashes 大小的 2 倍，因为它要存储 key 和 value
    Object[] mArray;
    //成员变量的个数，即容量
    int mSize;
    //遍历相关
    MapCollections<K, V> mCollections;
    //...
}
```

可以看到，ArrayMap 的实现是基于 2 个数组的：
一个 **int[] 数组 mHashes**，用于保存每个 item 的 hashCode.
一个 **Object[ ]数组 mArray**，保存 key/value 键值对。容量是上一个数组的两倍。
![arrayMap.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/450005/1572315403900-c5c3a084-b889-4410-9fc9-7134327f6465.jpeg#align=left&display=inline&height=314&originHeight=314&originWidth=955&size=26786&status=done&width=955)
并且为了减少频繁地创建和回收 Map 对象，ArrayMap 有缓存机制，分别缓存大小是 4 和 8 的 Map 对象。
其中 mSize 记录着该 ArrayMap 对象中有多少对数据，执行 put() 或者 append() 操作，则 mSize 会加 1，执行remove()，则 mSize 会减 1。mSize 往往小于 mHashes.length，如果 mSize 大于或等于 mHashes.length，则说明 mHashes 和 mArray 需要扩容。

在两个参数的构造函数中，先判断初始化容量：
如果小于 0，则给 mHashes 和 mArray 创建一个不可变的空ArrayMap；
如果等于 0，构建空的 mHashes mArray；
如果大于 0，则调用 allocArrays 进行分配空间初始化数组（扩容） ，mSize 初始化为 0。

# allocArrays

```java
private void allocArrays(final int size) {
    if (mHashes == EMPTY_IMMUTABLE_INTS) {
        throw new UnsupportedOperationException("ArrayMap is immutable");
    }
    //如果大小为 8
    if (size == (BASE_SIZE*2)) {
        synchronized (ArrayMap.class) {
            if (mTwiceBaseCache != null) { //查看有没有缓存
                final Object[] array = mTwiceBaseCache;
                mArray = array;  //从缓存池中取出 mArray
                mTwiceBaseCache = (Object[])array[0]; //将缓存池指向上一条缓存地址
                mHashes = (int[])array[1]; //从缓存中mHashes
                array[0] = array[1] = null; //清空缓存
                mTwiceBaseCacheSize--; //缓存池大小减 1
                if (DEBUG) Log.d(TAG, "Retrieving 2x cache " + mHashes
                                 + " now have " + mTwiceBaseCacheSize + " entries");
                return;
            }
        }
    } else if (size == BASE_SIZE) { //如果大小为 4
        synchronized (ArrayMap.class) {
            if (mBaseCache != null) { //看下有没有缓存，逻辑跟上面一样
                final Object[] array = mBaseCache;
                mArray = array;
                mBaseCache = (Object[])array[0];
                mHashes = (int[])array[1];
                array[0] = array[1] = null;
                mBaseCacheSize--;
                if (DEBUG) Log.d(TAG, "Retrieving 1x cache " + mHashes
                                 + " now have " + mBaseCacheSize + " entries");
                return;
            }
        }
    }
	//分配大小除了 4 和 8 之外的情况，其他则直接创建新的数组，mArray 的大小是 mHashes 的两倍
    mHashes = new int[size];
    mArray = new Object[size<<1];
}
```

当调用 allocArrays 分配内存，即扩容时，首先判断分配的大小是否等于 8 或者 4，如果等于，再判断时候存在缓存池，存在的话就重缓存中取出 mArray 和 mHashes。

从缓存池取出缓存的方式是将当前缓存池赋值给 mArray，将缓存池指向上一条缓存地址，将缓存池的第 1 个元素赋值为 mHashes，再把 mArray 的第 0 和第 1 个位置的数据置为 null，并将该缓存池大小执行减 1 操作。

其他情况则直接创建新的数组，mArray 的大小是 mHashes 的两倍。

# freeArrays

```java
private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
    if (hashes.length == (BASE_SIZE*2)) { //如果长度为 8
        synchronized (ArrayMap.class) {
            //如果当前缓存池数量小于10的话，则将其放入内存
            if (mTwiceBaseCacheSize < CACHE_SIZE) { 
                array[0] = mTwiceBaseCache; //array[0]指向原来的缓存池
                array[1] = hashes; 
                //清空 2+ 元素数据
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                //mTwiceBaseCache指向新加入缓存池的array
                mTwiceBaseCache = array;
                mTwiceBaseCacheSize++;
                if (DEBUG) Log.d(TAG, "Storing 2x cache " + array
                                 + " now have " + mTwiceBaseCacheSize + " entries");
            }
        }
    } else if (hashes.length == BASE_SIZE) { //原理同上
        synchronized (ArrayMap.class) {
            if (mBaseCacheSize < CACHE_SIZE) {
                array[0] = mBaseCache;
                array[1] = hashes;
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                mBaseCache = array;
                mBaseCacheSize++;
                if (DEBUG) Log.d(TAG, "Storing 1x cache " + array
                                 + " now have " + mBaseCacheSize + " entries");
            }
        }
    }
}
```

在 freeArrays 释放内存时，如果同时满足释放的 array 大小等于 4 或者 8，且相对应的缓冲池个数未达上限，则会把该 array 加入到缓存池中。
加入的方式是将数组 array 的第 0 个元素指向原有的缓存池，第 1 个元素指向 hashes 数组的地址，第 2 个元素以后的数据全部置为 null。再把缓存池的头部指向最新的 array 的位置，并将该缓存池大小执行加 1 操作。

# put

```java
public V put(K key, V value) {
    final int osize = mSize; //osize记录当前map大小
    final int hash;
    int index;
    //如果 key 为 null，则 hash 值为 0.
    if (key == null) { 
        hash = 0;
        index = indexOfNull(); //寻找 null 的下标
    } else {
        //mIdentityHashCode 默认为 false
        hash = mIdentityHashCode ? System.identityHashCode(key) : key.hashCode();
        //采用二分查找法，从 mHashes 数组中查找值等于 hash 的 key
        index = indexOf(key, hash);
    }
    //当 index 大于零，则代表的是从数据 mHashes 中找到相同的 key，
    //执行的操作等价于修改相应位置的 value
    if (index >= 0) {
        //index的2倍+1所对应的元素存在相应value的位置，具体可看上图
        index = (index<<1) + 1;
        final V old = (V)mArray[index];
        mArray[index] = value; //替换 value
        return old; //返回旧值
    }
	//否则 index 就是小于 0，当index<0，则代表是插入新元素，这里采用 ~运算，让 index 变成正数
    index = ~index;
    //如果当前 map 容量大于等于 mHashes 数组长度时，需要扩容
    if (osize >= mHashes.length) {
        //如果容量大于8，则扩容一半。
        //否则容量大于4，则扩容到8.
        //否则扩容到4
        final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
            : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);

        if (DEBUG) Log.d(TAG, "put: grow from " + mHashes.length + " to " + n);
		//临时保存 mHashes 和 mArray
        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        //重新根据 n 分配空间
        allocArrays(n);
        //由于ArrayMap并非线程安全的类，不允许并行，如果扩容过程其他线程调整mSize则抛出异常
        if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
            throw new ConcurrentModificationException();
        }
		//将原来老的数组（存在临时数组里）拷贝到新分配的数组
        if (mHashes.length > 0) {
            if (DEBUG) Log.d(TAG, "put: copy 0-" + osize + " to 0");
            System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
            System.arraycopy(oarray, 0, mArray, 0, oarray.length);
        }
		//释放临时数组空间
        freeArrays(ohashes, oarray, osize);
    }
	//如果 index 比 osize 小，代表 index 在数组中间，不需要扩容
    if (index < osize) {
        if (DEBUG) Log.d(TAG, "put: move " + index + "-" + (osize-index)
                         + " to " + (index+1));
        //需要将 index 位置后的数据通过拷贝往后移动一位，腾出中间的位置
        System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
        System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
    }

    if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
        if (osize != mSize || index >= mHashes.length) {
            throw new ConcurrentModificationException();
        }
    }
    //将 hash、key、value 添加相应数组的位置，数据个数mSize加1
    //hash 数组，就按照下标存哈希值
    //array 数组，根据下标，乘以 2 存 key，乘以 2+1 存 value
    mHashes[index] = hash;
    mArray[index<<1] = key;
    mArray[(index<<1)+1] = value;
    mSize++;
    return null;
}
```

put 方法在查找 index 下标是根据 key 和 hash，通过二分法查找的，关键方法 indexOf：

```java
int indexOf(Object key, int hash) {
    final int N = mSize;
    //如果集合大小为 0，则返回~0，即 -1
    if (N == 0) {
        return ~0;
    }
	//通过二分法查找 index
    int index = binarySearchHashes(mHashes, N, hash);
    //如果 index<0，说明该 hash 值之前没有存储过数据
    if (index < 0) {
        return index;
    }
    //如果index>=0,说明该hash值，之前存储过数据，找到对应的key，比对key是否相等。
    //相等的话，返回index。说明要替换。
    if (key.equals(mArray[index<<1])) {
        return index;
    }
    //以下两个 for 循环是在出现 hash 冲突的情况下，找到正确的 index 的过程：
    //从 index+1,遍历到数组末尾，找到 hash 值相等，且 key 相等的位置，返回
    int end;
    for (end = index + 1; end < N && mHashes[end] == hash; end++) {
        if (key.equals(mArray[end << 1])) return end;
    }
    //从index-1,遍历到数组头，找到hash值相等，且key相等的位置，返回
    for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
        if (key.equals(mArray[i << 1])) return i;
    }
    //key没有找到，返回一个负数(-1)。代表应该插入的位置
    return ~end;
}
```

**二分法 binarySearchHashes：**

```java
private static int binarySearchHashes(int[] hashes, int N, int hash) {
    try {
        return ContainerHelpers.binarySearch(hashes, N, hash);
    } catch (ArrayIndexOutOfBoundsException e) {
        if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
            throw new ConcurrentModificationException();
        } else {
            throw e; // the cache is poisoned at this point, there's not much we can do
        }
    }
}
//二分法算法实现
static int binarySearch(int[] array, int size, int value) {
    int lo = 0;
    int hi = size - 1;

    while (lo <= hi) {
        final int mid = (lo + hi) >>> 1;
        final int midVal = array[mid];

        if (midVal < value) {
            lo = mid + 1;
        } else if (midVal > value) {
            hi = mid - 1;
        } else {
            return mid;  // value found
        }
    }
    return ~lo;  // value not present
}
```
# 
# 
# putAll

```java
public void putAll(ArrayMap<? extends K, ? extends V> array) {
    final int N = array.mSize; //保存传进来的 array 大小
    ensureCapacity(mSize + N); //确保空间容量足够大
    //如果原来的容量是 0，但是传进来的大小不是 0 的话
    if (mSize == 0) { 
        //复制 array 
        if (N > 0) {
            System.arraycopy(array.mHashes, 0, mHashes, 0, N);
            System.arraycopy(array.mArray, 0, mArray, 0, N<<1);
            mSize = N;
        }
    } else {
        //构造循环调用 put 逐个添加
        for (int i=0; i<N; i++) {
            put(array.keyAt(i), array.valueAt(i));
        }
    }
}
```

putAll 方法首先会调用 ensureCapacity 方法确保当前空间容量足够大，然后判断当前 ArrayMap 是否有值，如果是空的话，比如第一次使用，就直接复制 array 到当前 ArrayMap 中，否则调用 put 方法逐个添加。

**ensureCapacity**
**
```java
public void ensureCapacity(int minimumCapacity) {
    final int osize = mSize;
    if (mHashes.length < minimumCapacity) {
        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        allocArrays(minimumCapacity);
        if (mSize > 0) {
            System.arraycopy(ohashes, 0, mHashes, 0, osize);
            System.arraycopy(oarray, 0, mArray, 0, osize<<1);
        }
        freeArrays(ohashes, oarray, osize);
    }
    if (CONCURRENT_MODIFICATION_EXCEPTIONS && mSize != osize) {
        throw new ConcurrentModificationException();
    }
}
```

判断如果传进来的 size 比原来的容量大，即代表不够容量了，需要调用 allocArrays 扩容，同时如果原来大小大于 0，则复制旧的数据到新扩容后的数组里。然后调用 freeArrays 释放临时数组，逻辑跟 put 方法里面有点类似。

# append

```java
public void append(K key, V value) {
    int index = mSize;
    //获取 hash 值
    final int hash = key == null ? 0
        : (mIdentityHashCode ? System.identityHashCode(key) : key.hashCode());
    //使用 append 前必须保证 mHashes 的容量足够大，否则抛出异常
    if (index >= mHashes.length) {
        throw new IllegalStateException("Array is full");
    }
    //如果 index>0,即代表当数据需要插入到数组的中间，则调用 put 来完成
    if (index > 0 && mHashes[index-1] > hash) {
        RuntimeException e = new RuntimeException("here");
        e.fillInStackTrace();
        Log.w(TAG, "New hash " + hash
              + " is before end of array hash " + mHashes[index-1]
              + " at index " + index + " key " + key, e);
        put(key, value);
        return;
    }
    //否则，将数据添加到队尾
    mSize = index+1;
    mHashes[index] = hash;
    index <<= 1;
    mArray[index] = key;
    mArray[index+1] = value;
}
```

# remove

```java
public V remove(Object key) {
    final int index = indexOfKey(key);
    if (index >= 0) {
        return removeAt(index);
    }
    return null;
}
```

首先通过 indexOfKey 查找 index，indexOfKey 里面调用了 indexOf 方法，同样通过二分法去查找。如果找到则调用 removeAt，找不到就返回 null。

# removeAt

```java
public V removeAt(int index) {
    //根据 index 获取 value
    final Object old = mArray[(index << 1) + 1];
    final int osize = mSize;
    final int nsize;
    //判断之前集合大小是否小于等于1，如果是的话，那么删除后集合就变成空集合了
    if (osize <= 1) {
        // Now empty.
        if (DEBUG) Log.d(TAG, "remove: shrink from " + mHashes.length + " to 0");
        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        mHashes = EmptyArray.INT;
        mArray = EmptyArray.OBJECT;
        freeArrays(ohashes, oarray, osize);
        nsize = 0;
    } else {
        nsize = osize - 1; //集合长度减一
        //根据情况来做内存收缩，如果 mHashes 长度大于8，并且集合大小小于当前空间的1/3
        if (mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3) {
            // Shrunk enough to reduce size of arrays.  We don't allow it to
            // shrink smaller than (BASE_SIZE*2) to avoid flapping between
            // that and BASE_SIZE.
            //如果当前集合长度大于8，则n为当前集合长度的1.5倍。否则n为8.
            //n 为收缩后的 mHashes 长度
            final int n = osize > (BASE_SIZE*2) ? (osize + (osize>>1)) : (BASE_SIZE*2);

            if (DEBUG) Log.d(TAG, "remove: shrink from " + mHashes.length + " to " + n);
 		    //分配新的更小的空间（收缩操作）
            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;
            allocArrays(n);
			//禁止并发
            if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                throw new ConcurrentModificationException();
            }
			//因为执行了收缩操作，所以要将老数据复制到新数组中。
            if (index > 0) {
                if (DEBUG) Log.d(TAG, "remove: copy from 0-" + index + " to 0");
                System.arraycopy(ohashes, 0, mHashes, 0, index);
                System.arraycopy(oarray, 0, mArray, 0, index << 1);
            }
            //在复制的过程中，排除不复制当前要删除的元素即可。
            if (index < nsize) {
                if (DEBUG) Log.d(TAG, "remove: copy from " + (index+1) + "-" + nsize
                                 + " to " + index);
                System.arraycopy(ohashes, index + 1, mHashes, index, nsize - index);
                System.arraycopy(oarray, (index + 1) << 1, mArray, index << 1,
                                 (nsize - index) << 1);
            }
        } else {
            //不需要收缩，用复制操作去覆盖元素达到删除的目的。
            if (index < nsize) {
                if (DEBUG) Log.d(TAG, "remove: move " + (index+1) + "-" + nsize
                                 + " to " + index);
                System.arraycopy(mHashes, index + 1, mHashes, index, nsize - index);
                System.arraycopy(mArray, (index + 1) << 1, mArray, index << 1,
                                 (nsize - index) << 1);
            }
            //记得置空，以防内存泄漏（再将最后一个位置设置为null）
            mArray[nsize << 1] = null;
            mArray[(nsize << 1) + 1] = null;
        }
    }
    //禁止并发
    if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
        throw new ConcurrentModificationException();
    }
    //更新集合长度
    mSize = nsize;
    //返回删除的值
    return (V)old;
}
```

remove() 过程：
通过二分查找 key 的 index，再根据 index 来选择移除动作；
当被移除的是 ArrayMap 的最后一个元素，则释放该内存，否则只做移除操作，这时会根据容量收紧原则来决定是否要收紧，当需要收紧时会创建一个更小内存的容量。

# clear

```java
public void clear() {
    if (mSize > 0) {
        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        final int osize = mSize;
        mHashes = EmptyArray.INT;
        mArray = EmptyArray.OBJECT;
        mSize = 0;
        freeArrays(ohashes, oarray, osize);
    }
    if (CONCURRENT_MODIFICATION_EXCEPTIONS && mSize > 0) {
        throw new ConcurrentModificationException();
    }
}
```

当 mSize 大于 0 时才操作，直接赋一个空的数组并调用 freeArrays 方法清理空间。

# ArrayMap 缺陷

ArrayMap 在并发时极小几率出现崩溃，详情看 ：[http://gityuan.com/2019/01/13/arraymap/#%E4%B8%89arraymap%E7%BC%BA%E9%99%B7%E5%88%86%E6%9E%90](http://gityuan.com/2019/01/13/arraymap/#%E4%B8%89arraymap%E7%BC%BA%E9%99%B7%E5%88%86%E6%9E%90)

# get

```java
public V get(Object key) {
    final int index = indexOfKey(key);
    return index >= 0 ? (V)mArray[(index<<1)+1] : null;
}
```

get 方法根据二分法查找出 index，然后从数组 mArray 中获取 value。

# 总结

1. ArrayMap 是 android 特有的类，内部实现基于两个数组，与 HashMap 相比，效率更高。
2. ArrayMap 在查找 index 时用的是二分法查找，在数据量比较少时，相比 HashMap 更省内存。
3. 扩容时，会先判断在缓存中查找，如果容量大于8，则**扩容一半**
4. ArrayMap 有内存收缩功能，**根据元素数量和集合占用的空间情况**，判断是否要执行收缩操作，避免**空间浪费**
5. 在数据量不大的情况下，推荐使用 ArrayMap，因为 HashMap 需要创建一个额外对象来保存每一个放入 map 的 entry，且容量的利用率比 ArrayMap 低，整体更消耗内存。数量大小 1000 左右为上限。
6. ArrayMap 是线程不安全的，这一点 HashMap 比较好，具体坑可以看 **ArrayMap 缺陷** 这一点。
