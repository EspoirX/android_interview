LinkedHashMap 继承了 HashMap，实现了 Map 接口。

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
{
  //...
}
```

# 节点 LinkedHashMapEntry
LinkedHashMap 的节点封装成 LinkedHashMapEntry，它继承了 HasMap 的 节点 HashMap.Node：

```java
//双向链表的头结点
transient LinkedHashMapEntry<K,V> head;
//双向链表的尾结点
transient LinkedHashMapEntry<K,V> tail;

static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
    LinkedHashMapEntry<K,V> before, after;
    LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

LinkedHashMapEntry 在 Node 的基础上上扩展了一下，改成了一个**双向链表**。同时类里有两个成员变量 head 和 tail，分别指向**内部双向链表的表头、表尾**。

# 构造函数

```java
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

public LinkedHashMap() {
    super();
    accessOrder = false;
}

public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}

public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

构造函数的参数跟 HashMap 差不多，putMapEntries 是 HashMap 的方法。唯一区别是在构造函数中，就是增加了一个 **accessOrder** 参数。

```java

/**
  * The iteration ordering method for this linked hash map: <tt>true</tt>
  * for access-order, <tt>false</tt> for insertion-order.
  *
  * @serial
  */
final boolean accessOrder;
```

accessOrder  默认是 false，迭代时输出的顺序是**插入节点的顺序**。若为true，则输出的顺序是**按照访问节点的顺序**。
 **LruCach 的构建就是传入 true**。 即 false 为插入顺序，true 为访问顺序。

# 增
LinkedHashMap 没有重写任何 put 方法。但是它重写了创建新节点的 **newNode** 方法。

```java
//HashMap 的 newNode 方法
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
    return new Node<>(hash, key, value, next);
}

//LinkedHashMap 的 newNode 方法
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMapEntry<K,V> p =
        new LinkedHashMapEntry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}
```

在 HashMap 中，newNode 是在 putVal 方法中调用的。putVal 会在  putMapEntries 或者 put 方法中调用。
可以看到，在 LinkedHashMap 中，newNode 构建了新的 LinkedHashMapEntry 后，通过 **linkNodeLast **方法**将新节点链接在内部双向链表的尾部。**

```java
private void linkNodeLast(LinkedHashMapEntry<K,V> p) {
    LinkedHashMapEntry<K,V> last = tail; //尾节点
    tail = p;  
    if (last == null) //如果尾节点是空，即集合之前是空的
        head = p; //p就是头节点
    else {
        //将新节点连接在链表的尾部
        p.before = last; 
        last.after = p;
    }
}
```

HashMap 给 LinkedHashMap 留了三个空方法重写：

```java
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

# afterNodeInsertion

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMapEntry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

afterNodeInsertion 方法在新增节点插入之后回调 ， 根据 evict 和   判断是否需要删除最老插入的节点。如果实现LruCache 会用到这个方法。在判断条件中，removeEldestEntry 方法默认返回的是 false。所以不会删除节点，返回 true 代表要删除最早的节点。通常构建一个 LruCache 会在达到 Cache 的上限是返回 true。

# 删
LinkedHashMap 没有重写 remove 方法，但它重写了  afterNodeRemoval 方法。afterNodeRemoval 方法在 removeNode 中回调，removeNode 方法在 remove 中回调。

```java
void afterNodeRemoval(Node<K,V> e) { // unlink
    //在删除节点 e 时，同步将e从双向链表上删除
    LinkedHashMapEntry<K,V> p = (LinkedHashMapEntry<K,V>)e,
    b = p.before, 
    a = p.after;
    //待删除节点 p 的前置后置节点都置空
    p.before = p.after = null;
    //如果前置节点是 null，则现在的头结点应该是后置节点 a
    if (b == null)
        head = a;
    else //否则将前置节点 b 的后置节点指向 a
        b.after = a;
    //同理如果后置节点时 null ，则尾节点应是 b
    if (a == null)
        tail = b;
    else
        a.before = b;  //否则更新后置节点 a 的前置节点为 b
}
```

# 查
LinkedHashMap 重写了 get 和 getOrDefault 方法:

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}

public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return defaultValue;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

在 HashMap 的基础上，新加了逻辑如果 accessOrder 为 true，则调用 afterNodeAccess：


```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMapEntry<K,V> last;  //原尾节点
     //如果 accessOrder 是 true ，且原尾节点不等于 e
    if (accessOrder && (last = tail) != e) { 
        //节点 e 强转成双向链表节点 p
        LinkedHashMapEntry<K,V> p = (LinkedHashMapEntry<K,V>)e, 
        b = p.before,
        a = p.after;
        p.after = null;   //p 现在是尾节点， 后置节点一定是 null
         //如果 p 的前置节点是 null，则 p 以前是头结点，所以更新现在的头结点是 p 的后置节点 a
        if (b == null)
            head = a;
        else
            b.after = a; //否则更新 p 的前直接点 b 的后置节点为 a
        if (a != null)  //如果 p 的后置节点不是 null，则更新后置节点 a 的前置节点为 b
            a.before = b;
        else  //如果原本 p 的后置节点是 null，则 p 就是尾节点。 此时更新 last 的引用为 p 的前置节点 b
            last = b;
        if (last == null) //原本尾节点是 null 则链表中就一个节点
            head = p;
        else { //否则更新当前节点 p 的前置节点为 原尾节点 last，last 的后置节点是 p
            p.before = last;
            last.after = p;
        }
        tail = p; //尾节点的引用赋值成 p
        ++modCount;  //修改 modCount
    }
}
```

afterNodeAccess 会**将当前被访问到的节点 e，移动至内部的双向链表的尾部。**
值得注意的是，afterNodeAccess() 函数中，会修改 modCount，因此当你正在 accessOrder=true 的模式下，迭代 LinkedHashMap 时，如果同时查询访问数据，也会导致 fail-fast，因为迭代的顺序已经改变。

# containsValue
LinkedHashMap 重写了该方法，相比 HashMap 的实现，**更为高效**。因为在 HashMap 中，该方法是用两个 for 循环实现的：

```java
//LinkedHashMap
public boolean containsValue(Object value) {
    for (LinkedHashMapEntry<K,V> e = head; e != null; e = e.after) {
        V v = e.value;
        if (v == value || (value != null && value.equals(v)))
            return true;
    }
    return false;
}

//HashMap
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

为什么不重写 containsKey? 因为 containsKey 用的 hash 寻址，已经很高效。

# 遍历
LinkedHashMap 重写了 entrySet  方法：

```java
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new LinkedEntrySet()) : es;
}

final class LinkedEntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { LinkedHashMap.this.clear(); }
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new LinkedEntryIterator();
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
        return Spliterators.spliterator(this, Spliterator.SIZED |
                                        Spliterator.ORDERED |
                                        Spliterator.DISTINCT);
    }
    public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
        if (action == null)
            throw new NullPointerException();
        int mc = modCount;
        // Android-changed: Detect changes to modCount early.
        for (LinkedHashMapEntry<K,V> e = head; (e != null && mc == modCount); e = e.after)
            action.accept(e);
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }
}
```

跟 HashMap 类似，其实主要是看 LinkedEntryIterator：

```java
final class LinkedEntryIterator extends LinkedHashIterator implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}

abstract class LinkedHashIterator {
    //下一个节点
    LinkedHashMapEntry<K,V> next;
    //当前节点
    LinkedHashMapEntry<K,V> current;
    int expectedModCount;

    LinkedHashIterator() {
        //初始化时，next 为 LinkedHashMap 内部维护的双向链表的扁头
        next = head;
        //记录当前 modCount，以满足 fail-fast
        expectedModCount = modCount;
        //当前节点为 null
        current = null;
    }

    //判断 next 是否为 null，默认 next 是 head  表头
    public final boolean hasNext() {
        return next != null;
    }

    final LinkedHashMapEntry<K,V> nextNode() {
        //记录要返回的 e。
        LinkedHashMapEntry<K,V> e = next;
        //判断 fail-fast
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        //如果要返回的节点是 null，异常
        if (e == null)
            throw new NoSuchElementException();
        //更新当前节点为 e
        current = e;
        //更新下一个节点是 e 的后置节点
        next = e.after;
        //返回e
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount;
    }
}
```

LinkedEntryIterator 继承了 LinkedHashIterator。值得注意的就是：nextNode() 就是迭代器里的 next() 方法 。
该方法的实现可以看出，**迭代 LinkedHashMap，就是从内部维护的双链表的表头开始循环输出。**
**而双链表节点的顺序在 LinkedHashMap 的增、删、改、查时都会更新。以满足按照插入顺序输出，还是访问顺序输出。**
**
# 总结

1. LinkedHashMap 继承了 HashMap，仅重写了几个方法，以改变它迭代遍历时的顺序。这也是其与 HashMap相比最大的不同。
2. 在每次插入数据，或者访问、修改数据时，会增加节点、或调整链表的节点顺序。以决定迭代时输出的顺序。
3. accessOrder ，默认是 false，则迭代时输出的顺序是插入节点的顺序。若为 true，则输出的顺序是按照访问节点的顺序。为true时，可以在这基础之上构建一个LruCache.
4. LinkedHashMap 并没有重写任何 put 方法。但是其重写了构建新节点的 newNode() 方法。在每次构建新节点时，将新节点链接在内部双向链表的尾部。
5. accessOrder=true 的模式下，在 afterNodeAccess() 函数中，会将当前被访问到的节点 e，移动至内部的双向链表的尾部。值得注意的是，afterNodeAccess() 函数中，会修改 modCount,因此当你正在accessOrder=true 的模式下，迭代 LinkedHashMap 时，如果同时查询访问数据，也会导致 fail-fast，因为迭代的顺序已经改变。
6. nextNode() 就是迭代器里的next()方法 。
该方法的实现可以看出，迭代 LinkedHashMap，就是从内部维护的双链表的表头开始循环输出。而双链表节点的顺序在LinkedHashMap的增、删、改、查时都会更新。以满足按照插入顺序输出，还是访问顺序输出。
7. 还有一个小小的优化，重写了 containsValue() 方法，直接遍历内部链表去比对 value 值是否相等。
8. 和 HashMap 一样，LinkedHashMap  是线程不安全的，它运行 key 和 value 为 null。
