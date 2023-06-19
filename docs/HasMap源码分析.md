
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    //...
}
```

HasMap 继承了 AbstractMap 类，实现了 Map，Cloneable 和 Serializable 接口。
HasMap 是一个数组加链表的结构：

![hashmap.png](https://s2.loli.net/2023/06/19/XoTglvxJCbsRS25.png)

上面这个图片结构是 JDK 1.7 的，在 JDK1.8 后，HashMap 的结构再也不是数组加链表，而是数组加链表加红黑树：

![hashmap1.jpeg](https://s2.loli.net/2023/06/19/Lp1NA7f4W2xdZSl.jpg)

HasMap 是线程不安全的，允许 **key为null**，**value为null**。遍历时无序。
其底层数据结构是**数组**称之为**哈希桶**，每个**桶里面放的是链表**，链表中的**每个节点**，就是哈希表中的**每个元素**。

HashMap的源码中，充斥个各种**位运算代替常规运算**的地方，以提升效率。

# transient关键字
在 HashMap 中，会看到有些变量用到了这个关键字去修饰，这里解释一下这个关键字的作用。

我们都知道一个对象只要实现了 Serilizable 接口，这个对象就可以被序列化，java 的这种序列化模式为开发者提供了很多便利，我们可以不必关系具体序列化的过程，只要这个类实现了 Serilizable 接口，这个类的所有属性和方法都会自动序列化。
然而在实际开发过程中，我们可能会遇到这样的问题，**这个类的某些属性需要序列化，而其他属性不需要被序列化**，比如一些敏感信息(如密码)，为了安全起见，不希望在网络操作中被传输，这些信息对应的变量就可以加上 **transient** 关键字。
 
总结：

1. transient 修饰的变量不再被序列化
2. 一个静态变量不管是否被 transient 修饰，均不能被序列化
3. transient 关键字只能修饰变量，而不能修饰方法和类。
4. 一旦变量被 transient 修饰，变量将不再是对象持久化的一部分，该变量内容在序列化后无法获得访问。


# 链表节点Node

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash; //hash 值
    final K key;
    V value;
    Node<K,V> next; //链表下一个元素

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

1.  HasMap 每一个节点都封装成了 Node，里面存储了哈希值，Key，Value和下一个节点。明显，Node 是一个单链表结构。
2. hashCode 的取值是是将 key 的 hashCode 和 value 的 hashCode 通过**亦或操作**得到的。
3. setValue 方法是给当前 value 赋新值并且返回旧值。

# HasMap构造函数

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

```

HasMap 有四个构造函数。

1. 先看无参构造函数 **HasMap()，**在这个构造函数中给一个变量赋值，**loadFactor** 的意思是**加载因子**，它的默认值 **DEFAULT_LOAD_FACTOR 为 0.75f。加载因子的作用是用于计算哈希表元素数量的阈值。  threshold = 哈希桶.length * loadFactor;**
2. 两个参数的构造函数 **HashMap(int initialCapacity, float loadFactor)，**initialCapacity 为初始化容量，**MAXIMUM_CAPACITY** 的默认值是 **1<<30，即2的30次方。**在这个构造函数中，首先进行边界处理，如果指定的初始化容量大于最大值，则为最大值，loadFactor 加载因子也不能为负数。最后再通过 tableSizeFor 方法计算出 **阈值 threshold，即哈希桶长度。**

```java
static final int tableSizeFor(int cap) {
    //经过下面的 或 和位移 运算， n最终各位都是1。
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    //判断 n 是否越界，返回 2 的 n 次方作为 table（哈希桶）的阈值
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

这里采用了位运算，目的是提高效率，tableSizeFor 会根据期望容量 cap，返回 **2 的 n 次方形式的** 哈希桶的实际容量 length。 返回值一般会 >=cap。

3. 构造函数 **HashMap(int initialCapacity)** 为指定初始化容量的构造函数。
4. 构造函数 **HashMap(Map<? extends K, ? extends V> m)** 的作用是新建一个哈希表，同时将另一个 map m 里的所有元素加入表中。

# putMapEntries
看下最后一个构造函数中的 putMapEntries 方法。

```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    //拿到 Map 的长度  
    int s = m.size(); 
    //如果长度大于0才进行操作
    if (s > 0) {
        //如果当前表是空的
        if (table == null) { // pre-size
            //根据 m 的元素数量和当前表的加载因子，计算出新的阈值
            float ft = ((float)s / loadFactor) + 1.0F;
            //修正阈值的边界 不能超过MAXIMUM_CAPACITY
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            //如果新的阈值大于当前阈值
            if (t > threshold){
                //返回一个 >= 新的阈值 t 的 满足 2的n 次方的阈值
                threshold = tableSizeFor(t);
            }
        }
        else if (s > threshold) //如果当前表不为空，但是map的长度大于当前阈值
            resize(); //进行扩容
        //遍历 m 依次将元素加入当前表中。
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

第一个参数是另外一个 Map，第二个参数 evict 初始化时为 false，其他情况为 true。

1. 首先拿到传入 Map 的长度 s。
2. 判断长度，大于 0 才进行后续操作
3. **table** 的类型是 **Node<K,V>[]，可以理解为是一个表（哈希桶），它的长度始终是 2 的幂。** 先判断当前的表是否为空，如果空的话，根据传进来 Map 的长度 s 和当前加载因子 loadFactor 计算出一个新的阈值 ft。
4. 然后在一个三元表达式中判断 ft 的边界是否合法，然后把它赋值给 t。
5. 如果新计算处理的阈值 t 大于当前阈值 threshold，则调用 tableSizeFor 方法计算并返回一个大于等于 t 并满足 2 的 n 次方的阈值赋给 threshold。
6. 如果当前表 table 不为空并且 **s>threshold 即 Map 的长度大于当前阈值，代表着当前长度不够了，要调用 resize 方法进行扩容。**
7. 最后一步则是循环遍历 Map，依次调用 putVal 方法将元素加入当前表中。

# resize()
resize 方法是 HasMap 中一个比较重要的方法。它的作用是给 HasMap 扩容。**是 HasMap 的一个重点。**

```java
final Node<K,V>[] resize() {
    //新建临时变量 oldTab 保存一下当前的哈希桶。
    Node<K,V>[] oldTab = table;
    //获取到哈希桶的容量 oldCap
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //获取当前阈值 oldThr
    int oldThr = threshold;
    //新的哈希桶长度newCap 和新的阈值 newThr 默认值为 0
    int newCap, newThr = 0;
    //如果当前容量大于 0
    if (oldCap > 0) {
        //如果当前容量已经到达上限
        if (oldCap >= MAXIMUM_CAPACITY) {
            //则设置阈值是 2 的 31 次方 -1，同时返回当前的哈希桶，不再扩容
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //否则如果当前容量没达到上限，并且新的容量为旧的容量的两倍（新的容量同时也要小于上限），
        //而且还要旧的容量大于等于默认初始容量 16
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY){
            //那么新的阈值也等于旧的阈值的两倍
            newThr = oldThr << 1; // double threshold
        }
    }
    //如果 oldCap = 0，即当前表是空的，但是有阀值，代表是初始化时指定了容量、阈值的情况。
    //那么新表的容量 newCap 就等于旧的阀值。
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        //如果当前表是空的，而且也没有阈值。代表是初始化时没有任何容量/阈值参数的情况 
        //此时新表的容量 newCap 为默认值 16
        //新的阀值 newThr 为默认容量16 * 默认加载因子0.75f = 12
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //如果新的阈值是0，对应的是当前表是空的，但是有阈值的情况
    if (newThr == 0) {
        /根据新表容量 newCap 和 当前加载因子 求出新的阈值 ft
        float ft = (float)newCap * loadFactor;
        //对新阀值进行边界修复后赋值给 newThr
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    //更新当前阀值
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //计算完 newCap,newThr 后。根据新的容量 构建新的哈希桶 newTab
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab; //更新哈希桶引用
    //进行判断，如果旧的哈希桶有值，则把旧的值转移到新的上面，否则就直接返回新的哈希桶
    if (oldTab != null) {
        //遍历旧的哈希桶
        for (int j = 0; j < oldCap; ++j) {
            //遍历的当前节点 e
            Node<K,V> e;
            //取出节点 oldTab[j] 并赋值给 e 保存着，然后判断当前节点是否为空
            if ((e = oldTab[j]) != null) {
                //如果不为空，则赋为空，以便GC。
                oldTab[j] = null;
                //判断当前节点还有没有下一个节点，如果没有的话（即当前链表就只有一个元素）
                if (e.next == null){
                    //直接将这个元素放置在新的哈希桶里
                    //注意这里取下标 是用 哈希值 与 桶的长度-1 。 
                    //由于桶的长度是2的n次方，这么做其实是等于 一个模
                    newTab[e.hash & (newCap - 1)] = e;
                }
                //如果发生过哈希碰撞 ,而且是节点数超过8个，转化成了红黑树
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    //如果发生过哈希碰撞，节点数小于8个。
                    //则要根据链表上每个节点的哈希值，依次放入新哈希桶对应下标位置。
                    
                    //因为扩容是容量翻倍，所以原链表上的每个节点，
                    //现在可能存放在原来的下标，即low位， 或者扩容后的下标，即high位。
                    //high位 =  low位+原哈希桶容量
                    
                    //低位链表的头结点、尾节点
                    Node<K,V> loHead = null, loTail = null;
                    //高位链表的头节点、尾节点
                    Node<K,V> hiHead = null, hiTail = null;
                    //临时节点 存放e的下一个节点
                    Node<K,V> next;
                    //循环链表
                    do {
                        //e 的下一个节点赋值给 next
                        next = e.next;
                        //这里又是一个利用位运算 代替常规运算的高效点：
                        //利用哈希值 与 旧的容量，可以得到哈希值去模后，
                        //是大于等于 oldCap 还是小于 oldCap，
                        //等于 0 代表小于 oldCap，应该存放在低位，否则存放在高位
                        if ((e.hash & oldCap) == 0) {
                            //给头尾节点指针赋值
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //高位逻辑一样
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null); //循环直到链表结束
                    
                    //将低位链表存放在原index处，
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //将高位链表存放在新index处
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

resize 注释在代码中，这里再强调一下逻辑，resize 方法可以分为两部分去看，第一部分是分别计算出新的哈希桶的长度 newCap 和新的阀值 newThr，以便后面创建新的哈希桶。第二部分则为创建新的哈希桶，会对旧的哈希桶进行判断，如果有元素则把元素转移到新的哈希桶上面，否则就直接返回。如果发生生哈希冲突，即两个值插在相同的 index 下了，一般有两种方法，链表法和开放地址法，HashMap 采用链表法，即把冲突的两个值组成链表插在对应的 index 下。

```java
//如果当前容量大于 0
if (oldCap > 0) {
    //如果当前容量已经到达上限
    if (oldCap >= MAXIMUM_CAPACITY) {
        //则设置阈值是 2 的 31 次方 -1，同时返回当前的哈希桶，不再扩容
        threshold = Integer.MAX_VALUE;
        return oldTab;
    }
    //否则如果当前容量没达到上限，并且新的容量为旧的容量的两倍（新的容量同时也要小于上限），
    //而且还要旧的容量大于等于默认初始容量 16
    else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
             oldCap >= DEFAULT_INITIAL_CAPACITY){
        //那么新的阈值也等于旧的阈值的两倍
        newThr = oldThr << 1; // double threshold
    }
}
```

第一个判断是判断当前容量是否大于 0，如果大于 0 的话，再判断 当前容量是否已经达到了上限 **MAXIMUM_CAPACITY**。MAXIMUM_CAPACITY 的值是 1 << 30，即 2 的 30 次方。如果达到上限，则设置当前的阀值 threshold 大小为 Integer.MAX_VALUE，即 2 的 31 次方 -1。并且返回旧的哈希桶，不再扩容。
否则判断 **新的容量为旧的容量的两倍（新的容量同时也要小于上限）并且旧的容量大于等于默认初始容量 16，**那么新的阈值也等于旧的阈值的两倍。

然后第二个判断：
```java
else if (oldThr > 0) // initial capacity was placed in threshold
    newCap = oldThr;
```
即 oldCap = 0 ，并且 oldThr 大于 0 时，oldThr 是当前阀值 threshold，所以这里是**当前表是空的，但是有阀值，代表是初始化时指定了容量、阈值的情况。则新表的容量 newCap 就等于旧的阀值。**
 
第三个判断 else:
```java
else {               // zero initial threshold signifies using defaults
    newCap = DEFAULT_INITIAL_CAPACITY;
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
}
```
这个 else 对应着就是 **如果当前表是空的，而且也没有阈值。代表是初始化时没有任何容量/阈值参数的情况 。**
那么 此时新表的容量 newCap 为默认值 16。新的阀值 newThr 为默认容量16 * 默认加载因子0.75f = 12。

第四个判断：
```java
if (newThr == 0) {
    //根据新表容量 newCap 和 当前加载因子 求出新的阈值 ft
    float ft = (float)newCap * loadFactor;
    //对新阀值进行边界修复后赋值给 newThr
    newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
              (int)ft : Integer.MAX_VALUE);
}
```
这里判断了 newThr 是否为 0，进过一到三的判断和赋值，可以看出还有一些情况下 newThr 是没有赋值的。 就是**当前表是空的，但是有阈值的情况，这时候会根据新表容量 newCap 和 当前加载因子 求出新的阈值 ft，然后对新阀值进行越界判断后赋值给 newThr。**
 
**最后再把计算处理的 newThr ** **赋值给 threshold 更新当前的阀值。**
 
进过一系列判断计算后，newCap 和 newThr 都计算出来了，接下来就是第二部分的创建新的哈希桶。

```java
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
table = newTab;
```
创建新的哈希桶其实就是根据新的长度 newCap 新建一个数组。新建完之后更新当前哈希桶。

```java
if (oldTab != null) {
    for (int j = 0; j < oldCap; ++j) {
        Node<K,V> e;
        if ((e = oldTab[j]) != null) {
            oldTab[j] = null;
            if (e.next == null)
                newTab[e.hash & (newCap - 1)] = e;
            else if (e instanceof TreeNode)
                ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
            else { // preserve order
                Node<K,V> loHead = null, loTail = null;
                Node<K,V> hiHead = null, hiTail = null;
                Node<K,V> next;
                do {
                    next = e.next;
                    if ((e.hash & oldCap) == 0) {
                        if (loTail == null)
                            loHead = e;
                        else
                            loTail.next = e;
                        loTail = e;
                    }
                    else {
                        if (hiTail == null)
                            hiHead = e;
                        else
                            hiTail.next = e;
                        hiTail = e;
                    }
                } while ((e = next) != null);
                if (loTail != null) {
                    loTail.next = null;
                    newTab[j] = loHead;
                }
                if (hiTail != null) {
                    hiTail.next = null;
                    newTab[j + oldCap] = hiHead;
                }
            }
        }
    }
}
```
这个判断就是把旧哈希桶的值转移到新的上面。

1.  通过 **if ((e = oldTab[j]) != null)** 判断当前节点是否为 null，并且赋值给 e 保存着。
2. 如果不为 null，则清空旧哈希桶的值：**oldTab[j] = null，释放资源**
3. **if (e.next == null)  newTab[e.hash & (newCap - 1)] = e;** 通过判断当前节点有没有下一个节点，如果没有的话也就是说旧的哈希桶只有一个元素。那么直接把取出来的 e 保存在新的哈希桶中。

（**这里取下标 是用 哈希值 与 桶的长度-1 。 由于桶的长度是2的n次方，这么做其实是等于 一个模运算。但是效率更高）**

4. 如果发生过哈希碰撞 ,而且是节点数超过8个，转化成了红黑树：if (e instanceof TreeNode)
5. 如果发生过哈希碰撞，节点数小于8个，则根据链表上每个节点的哈希值，依次放入新哈希桶对应下标位置。

因为扩容是容量翻倍，所以原链表上的每个节点，现在可能存放在原来的下标，即 low 位， 或者扩容后的下标，即high位。high位 =  low位+原哈希桶容量：

![hashmap2.png](https://s2.loli.net/2023/06/19/7fShwIZgmOvQRpy.png)

# putVal
看回 putMapEntries 方法里面的 putVal 方法：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    //tab存放 当前的哈希桶， p用作临时链表节点  
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //赋值给变量并且判断当前哈希表是否为空
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length; //如果空的话就直接扩容，并把长度赋给 n
    
    //如果当前index的节点是空的，表示没有发生哈希碰撞。 
    //直接构建一个新节点 Node，存在 index 处即可。
    //这里再啰嗦一下，index 是利用 哈希值 & 哈希桶的长度-1，替代模运算
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        //否则 发生了哈希冲突。
        Node<K,V> e; K k;
        //如果哈希值相等，key也相等，则是覆盖 value 操作
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode) //红黑树操作
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //普通插入操作，遍历链表
            for (int binCount = 0; ; ++binCount) {
                //把下个节点赋值给 e，并判断是否为空，如果空，即遍历已经到了尾部，
                //则通过 newNode 方法新建一个节点添加到新的尾部。
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    ///如果追加节点后，链表数量 >=8，则转化为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //如果找到了要覆盖的节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //如果 e 不是null，说明有需要覆盖的节点，
        if (e != null) { // existing mapping for key
            //则覆盖节点值，并返回原oldValue
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e); //这是一个空实现的函数，用作LinkedHashMap重写使用。
            return oldValue;
        }
    }
    //如果执行到了这里，说明插入了一个新的节点，所以会修改modCount，以及返回null。
    ++modCount;
    //更新size，并判断是否需要扩容。
    if (++size > threshold)
        resize();
    //这是一个空实现的函数，用作LinkedHashMap重写使用。
    afterNodeInsertion(evict);
    return null;
}
```

putVal 的作用是插入一个元素，如果参数 onlyIfAbsent 为 true，那么不会覆盖相同 key 的值 value。

1.  判断当前的哈希表是否为空，如果空则要扩容让它变为不空，并保存容量给 n。
2. 通过判断当前的节点是否为空，如果空的话，直接创建一个新的 Node 节点添加进去，tab = table 这个让 tab 有了 table 的引用。
3. 如果不为空，那么插进去的时候就会存在哈希冲突了。这时候首先判断当前节点的哈希值和 key 值如果都和插进来的一样，那么就进行覆盖 value 操作。然后就是判断是否进行红黑树操作，和普通插入操作。
4. 在普通遍历操作里，遍历链表，先把**当前节点的**下一个节点赋值给 e 保存着，然后判断是否有下一个节点，没有的话，直接创建一个新的节点 Node，插入到下一个节点中，添加节点后，如果链表数量 >=8，则转化为红黑树。然后 break 跳出循环。
5. 如果下个节点的哈希值和插进来的一样，就直接 break 跳出，不操作。
6. 否则最后覆盖节点
7. 最后更新 modCount 和 size，并检查是否要进行扩容操作。
8. afterNodeAccess 和 afterNodeInsertion 是两个空实现的函数，用于 LinkedHashMap 重写。

```java
// Callbacks to allow LinkedHashMap post-actions
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

总结：

1. HasMap 的运算尽量都用**位运算**代替 ，更高效。 
2. 对于扩容导致需要新建数组存放更多元素时，除了要将老数组中的元素迁移过来，也记得将老数组中的引用置null，以便GC。
3. 取下标 是用 **哈希值 与运算 （桶的长度-1）：** **i = (n - 1) & hash**。 由于桶的长度是 2 的 n 次方，这么做其实是等于 一个**模运算**。但是**效率更高。**
4. 扩容时，如果发生过哈希碰撞，节点数小于8个。则要根据链表上每个节点的哈希值，依次放入新哈希桶对应下标位置。大于8个，则交给红黑树算法。
5. 因为扩容是容量翻倍，所以原链表上的每个节点，现在可能存放在原来的下标，即 low 位， 或者扩容后的下标，即 high 位。**high位 = low位+原哈希桶容量**
6. 利用**哈希值 与运算 旧的容量** ，**if ((e.hash & oldCap) == 0)，**可以得到哈希值去模后，是大于等于 oldCap 还是小于 oldCap，**等于0代表小于oldCap**，**应该存放在低位，否则存放在高位**。这里又是一个利用位运算 代替常规运算的高效点。
7. 如果追加节点后，链表数量 >=8，则转化为红黑树
8. 插入节点操作时，有一些空实现的函数，用作 LinkedHashMap 重写使用。

# put(K key, V value)
插入方法：

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

put 方法实际上是调用了 putVal 方法，它的 onlyIfAbsent 参数为 false，所以它会覆盖 value 值，所以 put 方法同时也有修改功能。

# hash()

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

可以看到 hash 值是根据 key 通过 hash 方法计算出来的。这个方法称为 **扰动函数。**  
key 的 hash 值，并不仅仅只是 key 对象的**hashCode()** 方法的返回值，还会经过**扰动函数**的扰动，**以使 hash 值更加均衡。**  
因为 hashCode() 是 int 类型，取值范围是40多亿，只要哈希函数映射的比较均匀松散，碰撞几率是很小的。  

但就算原本的 hashCode() 取得很好，每个 key 的 hashCode() 不同，但是由于 HashMap的 哈希桶的长度远比hash 取值范围小，默认是16，所以当对 hash 值以桶的长度取余，以找到存放该 key 的桶的下标时，由于取余是通过与操作完成的，会忽略 hash 值的高位。

因此只有 hashCode() 的低位参加运算，发生不同的 hash 值，但是得到的 index 相同的情况的几率会大大增加，这种情况称之为 hash碰撞。 即，碰撞率会增大。

扰动函数就是为了解决 hash 碰撞的。它会综合 hash 值高位和低位的特征，并存放在低位，因此在与运算时，相当于高低位一起参与了运算，以减少 hash 碰撞的概率。（在JDK8之前，扰动函数会扰动四次，JDK8简化了这个操作）

# putAll(Map<? extends K, ? extends V> m)

```java
public void putAll(Map<? extends K, ? extends V> m) {
    putMapEntries(m, true);
}
```

putAll 方法实际上是调用了  putMapEntries 方法。作用是批量增加数据。

# putIfAbsent(K key, V value)

```java
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}
```

putIfAbsent 方法也是插入数据，实际上也是调用了 putVal 方法，但它的 onlyIfAbsent 参数为 true，所以插入相同的 key 时不会覆盖 value。

# remove(Object key)

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```

根据 key 删除节点。如果key对应的value存在，则删除这个键值对。 并返回 value。如果不存在 返回 null。

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    //p 是待删除节点的前置节点
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    //如果哈希表不为空，则根据hash值算出的index下 有节点的话。
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        //node 是待删除节点
        Node<K,V> node = null, e; K k; V v;
        //如果链表头的就是需要删除的节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p; //将待删除节点引用赋给node
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                //否则循环遍历 找到待删除节点，赋值给node
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
         //如果有待删除节点 node，且 matchValue 为 false，或者值也相等
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p) //如果node ==  p，说明是链表头是待删除节点
                tab[index] = node.next; //node.next 为 null
            else //否则待删除节点在表中间
                p.next = node.next; //node.next 为下一个节点，即用下一个节点覆盖要删除的节点
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

从哈希表中删除某个节点， 如果参数 matchValue 是 true，则必须 key 、value 都相等才删除。如果 movable 参数是 false，在删除节点时，不移动其他节点。

**removeNode 方法主要的逻辑是找出待删除的节点 node，如果待删除的节点是链表头那么 node.next 的值即为 null，所以通过  tab[index] = node.next; ，tab 的值也会变成 null。**  
**如果待删除节点在链表中间，那么 node.next 的值就是下一个节点的值。通过 p.next = node.next; 操作就是相当于把节点往前挪一位，覆盖点待删除的节点。**  
 
afterNodeRemoval 是一个空方法，LinkedHashMap 重写用的回调函数。

# remove(Object key, Object value)

```java
public boolean remove(Object key, Object value) {
    return removeNode(hash(key), key, value, true, true) != null;
}
```

根据 key 和 value 去删除节点。可以看到也是调用了 removeNode 方法。但是 matchValue 参数为 true。
# 
get(Object key) 

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```
根据 key 查找节点，传入扰动后的哈希值 和 key 找到目标节点Node，调用了 getNode 方法。

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    //判断哈希表是否为空，并且把结果的前置节点赋值给 first
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //如果 hash 值和 key 相等，直接返回 first
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //否则判断下一个节点是否为空，并赋值给 e
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //循环遍历查找，找到就返回 e。
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

getNote 逻辑基本跟删除的差不多，注释在代码上。

# getOrDefault(Object key, V defaultValue)

```java
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
```
getOrDefault 方法跟 get 方法一样，只不过如果找不到就返回一个默认值。

# containsKey(Object key)

```java
public boolean containsKey(Object key) {
    return getNode(hash(key), key) != null;
}
```
该方法判断是否存在某个 key，实际上是调用了 getNode 查找方法，判断能不能找到。

# containsValue(Object value)

```java
public boolean containsValue(Object value) {
    Node<K,V>[] tab; V v;
    if ((tab = table) != null && size > 0) {
        for (int i = 0; i < tab.length; ++i) {
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                if ((v = e.value) == value ||
                    (value != null && value.equals(v)))
                    return true;
            }
        }
    }
    return false;
}
```
查找是否存在某个 value，可以看到它的逻辑就是遍历当前哈希表。然后判断 value 是否相等，找到即返回 true。

# HasMap 遍历

```java
transient Set<Map.Entry<K,V>> entrySet;

public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}
```

遍历用到了 entrySet 方法。可以看到这个方法获取的是 EntrySet 实例，EntrySet 是 HasMap 的一个内部类：


```java
final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }
    public final boolean contains(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>) o;
        Object key = e.getKey();
        Node<K,V> candidate = getNode(hash(key), key);
        return candidate != null && candidate.equals(e);
    }
    public final boolean remove(Object o) {
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Object value = e.getValue();
            return removeNode(hash(key), key, value, true, true) != null;
        }
        return false;
    }
    public final Spliterator<Map.Entry<K,V>> spliterator() {
        return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            // Android-changed: Detect changes to modCount early.
            for (int i = 0; (i < tab.length && modCount == mc); ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

可以看到，在 EntrySet 方法中，remove 和 contains 方法的最终实现还是调用了 removeNode 和 getNode。  
iterator 方法返回的是一个 EntryIterator。平时使用遍历也是获取这个东西最多：

```java
final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
   public final Map.Entry<K,V> next() { return nextNode(); }
}
```

EntryIterator 继承了 HashIterator:

```java
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    HashIterator() {
        //因为hashmap也是线程不安全的，所以要保存modCount。用于fail-fast策略
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        //next 初始时，指向 哈希桶上第一个不为null的链表头
        if (t != null && size > 0) { // advance to first entry
            do {} while (index < t.length && (next = t[index++]) == null);
        }
    }

    public final boolean hasNext() {
        return next != null;
    }

    //由这个方法可以看出，遍历HashMap时，顺序是按照哈希桶从低到高，
    //链表从前往后，依次遍历的。属于无序集合。
    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        //fail-fast策略
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        //依次取链表下一个节点，
        if ((next = (current = e).next) == null && (t = table) != null) {
            //如果当前链表节点遍历完了，则取哈希桶下一个不为null的链表头
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        //fail-fast策略
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        //最终还是利用removeNode 删除节点
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount;
    }
}
```

nextNode 方法的遍历顺序是从哈希桶从低到高，链表从前到后，它不会根据节点内容去排序，所以说 HasMap 是无序的。



参考文献：[https://blog.csdn.net/zxt0601/article/details/77413921](https://blog.csdn.net/zxt0601/article/details/77413921)
