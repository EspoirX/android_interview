实现了 Cloneable 接口：

```java
public class SparseArray<E> implements Cloneable {
	//...
}
```

# 成员变量

```java
public class SparseArray<E> implements Cloneable {
    //DELETE 标记
    private static final Object DELETED = new Object();
    //用于标记当前是否有待垃圾回收(GC)的元素
    private boolean mGarbage = false;
	//存储 key 的数组
    private int[] mKeys;
    //存储 value 的数组
    private Object[] mValues;
    //集合元素大小
    private int mSize;
}
```

可以看到，SparseArray 也是基于 2 个数组实现的，其中，**key 指定为 int 类型。**

# 构造方法

```java
public SparseArray() {
    this(10);
}

public SparseArray(int initialCapacity) {
    if (initialCapacity == 0) {
        mKeys = EmptyArray.INT;
        mValues = EmptyArray.OBJECT;
    } else {
        mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
        mKeys = new int[mValues.length];
    }
    mSize = 0;
}
```

构造方法的参数是初始化数组大小 initialCapacity，默认值为 10，可以看到当 initialCapacity 为 0 时，mKeys 和 mValues 为空数组，否则 mValues 通过 ArrayUtils#newUnpaddedObjectArray 方法创建。

```java
public static Object[] newUnpaddedObjectArray(int minLen) {
    return (Object[])VMRuntime.getRuntime().newUnpaddedArray(Object.class, minLen);
}
```

newUnpaddedObjectArray 通过 VMRuntime 的 newUnpaddedArray 方法创建数组，但是数组的大小不定，最小为 minLen。
> 返回至少为minLength，但可能更大的数组。 增加的大小来自避免在数组之后进行任何填充。 填充量取决于componentType和内存分配器实现而变化。
> 
> Java数据类型在内存中使用不同的大小。 如果仅分配所需的容量，则在下一次内存分配之前，末端的一些空间将不使用。 此空间称为填充。 因此，为了不浪费此空间，将数组做大一点。


所以初始化时，SparseArray 的初始化空间大小**大于等于10。**

**然后 mKeys 的长度跟 mValues 一样。**
最后集合大小初始化为 0。
![SparseArray.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/450005/1572401819012-32f138c4-0d09-41bf-ba91-32c2df0e3d8b.jpeg#align=left&display=inline&height=238&originHeight=238&originWidth=956&size=17024&status=done&width=956)

# get

```java
public E get(int key) {
    return get(key, null);
}

public E get(int key, E valueIfKeyNotFound) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i < 0 || mValues[i] == DELETED) {
        return valueIfKeyNotFound;
    } else {
        return (E) mValues[i];
    }
}
```

1. get 方法有两个重载，可以指定如果找不到的话返回一个默认值 valueIfKeyNotFound。
2. 在 get 方法中，下标 i 是通过二分法查找的， ContainerHelpers#binarySearch 方法是二分法的实现。
3. 然后直接根据下标在 mValues 数组中取值，如果找不到下标或者该 value 被标记成 DELETED 的话就返回 valueIfKeyNotFound。

# delete

```java
public void delete(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}
```

delete 方法首先也是通过二分法查找出下标 i。
如果 i>=0 代表找到，然后判断当前下标对应的 value 是否被标记成 DELETED，如果不是，则标记一下，然后修改 mGarbage 变量。

# removeReturnOld

```java
public E removeReturnOld(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            final E old = (E) mValues[i];
            mValues[i] = DELETED;
            mGarbage = true;
            return old;
        }
    }
    return null;
}
```

removeReturnOld 跟 delete 方法其实逻辑是一样的，唯一区别是 removeReturnOld 会返回已经删除的 value 值。

# remove

```java
public void remove(int key) {
    delete(key);
}
```

直接调用 delete 方法。

# removeAt

```java
public void removeAt(int index) {
    if (mValues[index] != DELETED) {
        mValues[index] = DELETED;
        mGarbage = true;
    }
}
```

因为 SparseArray 的 key 指定是 int 类型，所以如果已经知道了下标 key，那么可以直接根据 key 来删除，删除逻辑跟 delete 方法一致，这样就免去了二分查找了。

# gc

```java
private void gc() {
    int n = mSize;
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;

    for (int i = 0; i < n; i++) {
        Object val = values[i];

        if (val != DELETED) {
            if (i != o) {
                keys[o] = keys[i];
                values[o] = val;
                values[i] = null;
            }

            o++;
        }
    }

    mGarbage = false;
    mSize = o;
}
```

因为 SparseArray 中可能出现只移除 value 和 value 两者之一的情况，所以此方法就用于移除无用的引用，这里主要逻辑是在 for 循环里面：

1. 在 delete 方法中，被删除的 value 会被标记成 DELETED，所以正常情况下第一个 if 条件都会被满足，**这时候 i 和 o 还是相等的。**
2. 当遍历到被删除的 value 时，因为 value 被标记成了 DELETED，所以条件不成立，执行下一次循环。
3. **因为 o 是在 if 条件下自加一的**，**所以在步骤 2 之后，i 的大小已经比 o 大了**，所以如果再遇到正常的 value 时，第二个 if 条件就会成立。
4. 在第二个 if 条件中，索引 i 处的值赋值到索引 o 处，然后释放 i 处的空间。相当于向前移了一位。
5. 遍历结束后，修改 mGarbage 标记，更新 mSize 大小。

# put

```java
public void put(int key, E value) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        mValues[i] = value; //找到则覆盖
    } else {
        i = ~i;

        if (i < mSize && mValues[i] == DELETED) {
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }

        if (mGarbage && mSize >= mKeys.length) {
            gc();

            // Search again because indices may have changed.
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }

        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```

1. 首先通过二分法查找下标 i。
2. 如果 i 大于等于 0，代表找到，即原来在 i 的位置上存在 value，那么直接进行覆盖操作。
3. 如果小于 0，先取反转变为正数，判断如果 i 没有越界并且标记成 DELETED 的话，则直接复用 DELETED 。
4. 如果 mGarbage 为 true，并且当前集合大小大于 mKeys 的长度，这里应该是代表有元素被删除了，还是需要扩容？ 先进行 gc 操作。
5. gc后，下标 i 可能发生变化，所以再次用二分查找找到应该插入的下标 i 
6. 通过复制或者扩容数组，将数据存放到数组中
7. 最后 mSize 加一

# GrowingArrayUtils#insert

GrowingArrayUtils 源码地址：

```java
public static int[] insert(int[] array, int currentSize, int index, int element) {
   //断言 确认 当前集合长度 小于等于 array数组长度
    assert currentSize <= array.length;
    //如果不需要扩容
    if (currentSize + 1 <= array.length) {
        //将array数组内元素，从index开始 后移一位
        System.arraycopy(array, index, array, index + 1, currentSize - index);
        //在index处赋值
        array[index] = element;
        //返回
        return array;
    }
    //需要扩容，构建新的数组
    int[] newArray = ArrayUtils.newUnpaddedIntArray(growSize(currentSize));
    //将原数组中index之前的数据复制到新数组中
    System.arraycopy(array, 0, newArray, 0, index);
    //在index处赋值
    newArray[index] = element;
    //将原数组中index及其之后的数据赋值到新数组中
    System.arraycopy(array, index, newArray, index + 1, array.length - index);
    //返回
    return newArray;
}
```

# 总结

1. SparseArray 结构也是采用两个数组完成的，但相对于 ArrayMap 简单，SparseArray 的 mKeys 和 mValues 肯存在不对齐的情况，所以需要 gc 操作。
2. SparseArray 是指定 key 类型为 int，Android sdk中，还提供了三个类似思想的集合：SparseBooleanArray（key 为 int，value 为 boolean），SparseIntArray（key 和 value 为 int），SparseLongArray（key 为 int，value 为 long）
3. SparseArray 在数据量小的情况下比较节省内存，如果 key 为 int 类型的集合，推荐使用，但数据量大时不推荐，原因和 ArrayMap 类似。
4. SparseArray 查找下标 index 时跟 ArrayMap 一样采用二分法查找。
5. SparseArray 的 delete 方法并不是马上删除，而是把待删除的节点标记成 DELETED，在 put 时，可以再利用，在 gc 时才删除，这样可以达到充分利用的效果，避免多余的创建和删除操作。

# HashMap，Hashtable，LinkedHashMap，ArrayMap，SparseArray 总结

**数据结构**
**

- HashMap 和 LinkedHashMap 采用的是数组+链表+红黑树
- Hashtable 采用的是数组+链表
- ArrayMap 和 SparseArray 采用的都是两个数组，Android 专门针对内存优化而设计的

**内存优化**
**

- ArrayMap 比 HashMap 更节省内存，综合性能方面在数据量不大的情况下，推荐使用 ArrayMap；
- HashMap 需要创建一个额外对象来保存每一个放入 map 的 entry，且容量的利用率比 ArrayMap 低，整体更消耗内存
- SparseArray 比 ArrayMap 节省 1/3 的内存，但 SparseArray 只能用于 key 为 int 类型的 Map，所以 int  类型的 Map 数据推荐使用 SparseArray；


**性能方面**


- HashMap 和 LinkedHashMap 查找下标使用的是 hash key 寻址**（i = (n - 1) & hash)**；Hashtable 采用的是**  int index = (hash & 0x7FFFFFFF) % tab.length; **ArrayMap 和 SparseArray 采用的都是**二分法，**数据量大的情况下 HashMap 效率最高。
- ArrayMap 查找时间复杂度 O(logN)；ArrayMap 增加、删除操作需要移动成员，速度相比较慢，对于个数小于**1000** 的情况下，性能基本没有明显差异。
- HashMap 查找、修改的时间复杂度为 O(1)；
- SparseArray 适合频繁删除和插入来回执行的场景，性能比较好。


**缓存机制**
**

- ArrayMap 针对容量为 4 和 8 的对象进行缓存，可避免频繁创建对象而分配内存与 GC 操作，这两个缓存池大小的上限为 10 个，防止缓存池无限增大；
- HashMap 和 Hashtable 没有缓存机制
- SparseArray 有延迟回收机制，提供删除效率，同时减少数组成员来回拷贝的次数


**扩容机制**
**

- ArrayMap 是在容量满的时机触发容量扩大至原来的 1.5 倍，在容量不足 1/3 时触发内存收缩至原来的 0.5 倍，更节省的内存扩容机制
- HashMap 是在容量的 0.75 倍时触发容量扩大至原来的 2 倍，且没有内存收缩机制。HashMap 扩容过程有hash 重建，相对耗时。所以能大致知道数据量，可指定创建指定容量的对象，能减少性能浪费。
- HashMap 扩容触发条件是当发生哈希冲突，并且当前实际键值对个数是否大于或等于阈值 threshold，默认为0.75*capacity；
- HashMap 扩容操作是针对哈希表 table 来分配内存空间，每次扩容是至少是当前大小的 2 倍，扩容的大小一定是 2^n， 另外，扩容后还需要将原来的数据都 transfer 到新的 table，这是耗时操作。

**并发问题**
**

- ArrayMap 是非线程安全的类，大量方法中通过对 mSize 判断是否发生并发，来决定抛出异常。但没有覆盖到所有并发场景，比如大小没有改变而成员内容改变的情况就没有覆盖。
- HashMap 是在每次增加、删除、清空操作的过程将 modCount 加 1，在关键方法内进入时记录当前 mCount，执行完核心逻辑后，再检测 mCount 是否被其他线程修改，来决定抛出异常。这一点的处理比 ArrayMap 更有全面。
- Hashtable 是线程安全的
