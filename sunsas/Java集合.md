# 1.1 Java集合

> 作者：SunSAS
>
> **介绍：** SunSAS是SunSAS

## 1.1.1 ArrayList

### ArrayList简介
ArrayList 的底层是数组队列，相当于动态数组。与 Java 中的数组相比，它的容量能动态增长。在添加大量元素前，应用程序可以使用`ensureCapacity`操作来增加 ArrayList 实例的容量。这可以减少递增式再分配的数量。

![ArrayList类结构](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/picgo2022image-20220215153405264.png)

它继承于 AbstractList，实现了 List, RandomAccess, Cloneable, java.io.Serializable 这些接口。

注意它实现了**RandomAccess**接口，而这个接口里面是什么也没有的。只是一个标识接口，表明实现这个这个接口的 List 集合是支持**快速随机访问**的

ArrayList线程不安全，多线程应该使用**vector**

### ArrayList源码

#### 属性
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     空数组
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     用于默认大小空实例的共享空数组实例。
     我们把它从EMPTY_ELEMENTDATA数组中区分出来，以知道在添加第一个元素时容量需要增加多少。
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     保存ArrayList数据的数组。
     ArrayList的容量是这个数组缓冲区的长度。
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
    
    /**
     * The maximum size of array to allocate.
     数组最大大小
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
}    
```
#### 构造方法
```java
/**
     * Constructs an empty list with the specified initial capacity.
     *带初始容量参数的构造函数
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * Constructs an empty list with an initial capacity of ten.
     默认无参构造函数
     初始其实是空数组 当添加第一个元素的时候数组容量才变成10
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * Constructs a list containing the elements of the specified
     构造一个包含指定集合的元素的列表，按照它们由集合的迭代器返回的顺序。
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

```

#### 重要方法
**add**
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```
**判断检查**

```java
private void ensureCapacityInternal(int minCapacity) {
    //这里就是初始化添加第一个元素时的逻辑
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        //获取默认容量10与传入的minCapacity比较，取最大值
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    //判断是否要扩容
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    //这个minCapacity就是数组元素个数+1
    //如果数组元素超过数组的大小（就是没有空位了）
    if (minCapacity - elementData.length > 0)
        //扩容
        grow(minCapacity);
}
// 检查最大容量，扩容时用到
private static int hugeCapacity(int minCapacity) {
    //如果ArrayList长度已经越界，抛出异常
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    // 否则取一个最大值
    // MAX_ARRAY_SIZE = Integer.MAX_VALUE -8
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```
上面代码可知数组是在已满的情况下再次添加才会扩容。

**扩容**

```java
private void grow(int minCapacity) {
    // elementData是保存ArrayList数据的数组
    // 得到数组的长度就是旧容量
    int oldCapacity = elementData.length;
    // 新容量是在旧容量基础扩大1.5倍（这里是位运算）
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 再检查新容量是否超出了ArrayList所定义的最大容量
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        // 若超出了，则调用hugeCapacity()来比较minCapacity和 MAX_ARRAY_SIZE
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    // 然后就copy数据了
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**复制**

上面的`Arrays.copyOf(elementData, newCapacity)`是`Array`下的方法
```java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
```
其中调用了`System.arraycopy()`这是native方法。就是copy一个数组到你的目标数组中去。





***
## 1.1.2 LinkedList

### 简介
LinkedList是一个实现了List接口和Deque接口的双端链表。 LinkedList底层的链表结构使它支持高效的插入和删除操作。

![LinkedList类结构图](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/picgo2022image-20220215165351540.png)

`LinkedList`与`ArrayList`在于前者底层使用链表，而后者底层使用数组，这也决定了`LinkedList`随机访问缓慢，插入删除快,适合用来实现Stack(堆栈)与Queue(队列)；ArrayList随机访问快，插入删除慢。

![LinkedList链表结构图](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/1515111-20190530230420117-1509500427.png)

其链表为内部类Node

```java
private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

### 源码
#### 属性

```
transient int size = 0;

/**
 * Pointer to first node.
 */
transient Node<E> first;

/**
 * Pointer to last node.
 */
transient Node<E> last;
```

#### 重要方法

**add**
```java
public boolean add(E e) {
    linkLast(e);
    return true;
}
public void addFirst(E e) {
    linkFirst(e);
}

public void addLast(E e) {
    linkLast(e);
}

void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```
**remove**
```java
// 删除指定值的node
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
// 根据索引删除
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
public E remove() {
    return removeFirst();
}
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
```

其它常用APi都没必要讲了。





---
## 1.1.3 HashMap

### 简介
`HashMap`用来存放键值对，基于哈希表的`Map`接口实现，是常用的Java集合之一。与`HashTable`主要区别为不支持同步和**允许null作为key和value**。但是并发也不会用到`HashTable`，因为它仅仅是把`HashMap`的方法加上`synchronized`关键字而已，并发效率并较低。应该使用ConcurrentHashMap解决并发。

HashMap在面试中经常问道，原因是它可以衍生很多问题，例如HashMap为什么**线程不安全**，HashMap与HashTable的区别，**底层实现**是怎样，**红黑树**了不了解，**ConcurrentHashMap**为什么是线程安全的？在JDk7，JDK8中有什么变化？

### 底层实现
**JDK1.8之前**  
HashMap底层是**数组**和**链表**结合在一起使用。

![JDK1.8前](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/16240dbcc303d872.png)

因为HashMap需要计算`hash`值，存入对应位置。但是难免会有**冲突**（hashcode相同）这时候就需要解决冲突，可以再散列，也可以像这里一样，遇到哈希冲突，则将冲突的值加到链表中即可。

**JDK1.8之后**  
如果冲突过多，可能导致链表过长，这时就导致它的查询效率降低。所以jdk1.8做了改进，当链表长度大于阈值（默认为8）时，将链表转化为**红黑树**，以减少搜索时间。 
>红黑树是一种自平衡排序二叉树，TreeSet，TreeMap，HashMap中都使用了红黑树。

### 源码
#### 属性

```java
//默认的初始容量是16，必须是2的次方
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
//最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;

static final float DEFAULT_LOAD_FACTOR = 0.75f;

static final int TREEIFY_THRESHOLD = 8;
// 当桶(bucket)上的结点数小于这个值时树转链表
static final int UNTREEIFY_THRESHOLD = 6;
// 存储元素的数组，总是2的幂次倍
static final int MIN_TREEIFY_CAPACITY = 64;

/* ---------------- Fields -------------- */
// 存储元素的数组，总是2的幂次倍
transient Node<K,V>[] table;
// 存放具体元素的集
transient Set<Map.Entry<K,V>> entrySet;
// 存放元素的个数，并非数组的长度。
transient int size;
// 每次扩容和更改map结构的计数器
transient int modCount;
// 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
int threshold;
// 负载因子，默认是0.75。元素个数占了数组的75%就会扩容。不能设得太小，一般也不会低于0.5，否则空间使用率太低。也不会设得很高（不高于0.8），否则冲突过多。
final float loadFactor;
```
> 关于负载因子，参考[为什么 HashMap 的加载因子是0.75？](https://mp.weixin.qq.com/s/LKSNpAW5G6JqZIGzf7979w)  
> 不过他说到的Hashmap使用了开放地址法+拉链法应该不对，哪有使用开放地址？

#### 内部类
**Node**  

```java
static class Node<K,V> implements Map.Entry<K,V> {
        // 哈希值，存放元素到hashmap中时用来与其他元素hash值比较
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
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
**TreeNode** 

红黑树节点部分源码（太多了） 
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // 父
        TreeNode<K,V> left;    // 左
        TreeNode<K,V> right;   // 右
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;           // 判断颜色
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }

```

#### 构造方法

```java
// 无参构造
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
// 包含另一个“Map”的构造函数，将m的所有元素存入本HashMap实例中
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
// 可以指定大小
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
// 指定“容量大小”和“加载因子”的构造函数 
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
```
> 注意到table并没有初始化，table在`resize()`才初始化。

关注一下`tableSizeFor()`这个方法
```java
/**
 * Returns a power of two size for the given target capacity.
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
`tableSizeFor()`的主要功能是返回一个比给定整数大且最接近的2的幂次方整数。比如initialCapacity为30，则返回32。

萌新看到这些代码难免一脸懵逼，反正我是没看懂。看了别人的介绍才知道是这么弄得。[HashMap中的tableSizeFor方法](https://blog.csdn.net/Z_ChenChen/article/details/82955221)

```java
n= 1000 0000  0000 0000  0000 0000  0000 0000
n |= n >>> 1;  1100 0000  0000 0000  0000 0000  0000 0000  将最高位拷贝到下1位
n |= n >>> 2;  1111 0000  0000 0000  0000 0000  0000 0000  将上述2位拷贝到紧接着的2位
n |= n >>> 4;  1111 1111  0000 0000  0000 0000  0000 0000  将上述4位拷贝到紧接着的4位
n |= n >>> 8;  1111 1111  1111 1111  0000 0000  0000 0000  将上述8位拷贝到紧接着的8位
n |= n >>> 16; 1111 1111  1111 1111  1111 1111  1111 1111  将上述16位拷贝到紧接着的16位
```
无论是什么数（在int范围内）经过这几次运算，能够保证他最高位之后都是1。
引用[HashMap方法hash()、tableSizeFor()](https://blog.csdn.net/huzhigenlaohu/article/details/51802457)的一幅图：
![示例](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20160408183651111.png)
注意到`initialCapacity`可以为0，如果小于0直接抛出异常。所以n可能会等于-1.不过以无所谓，因为这里做了`或`运算，结果n还是-1，最后则返回1。

#### 重要方法
**put()**

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```
`put()`调用的内部`putVal()`,此方法为默认修饰符，只有同包可以访问。传入hash参数由`hash()`方法生成  

**hash()**
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
得出hash值方法是：调用`Object.hashCode()`产生了`hash`值，与其右移16位做**亦或**运算。  
实际上是key的**hash值高16位不变，低16位与高16位异或作为key的最终hash值。**（h >>> 16，表示无符号右移16位，高位补0，任何数跟0异或都是其本身，因此key的hash值高16位不变。）

HashMap中计算下标需要用到hash值，也就是下面代码`putVal()`也出现的方法。
```java
index = (n - 1) & hash //n是数组长度
```
![hash](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20160408155102734.png)  
由于低n位用来计算index值很频繁。而高位在数组长度比较小时没用上。所以对低16位进行了扰乱，让高位也参与到运算中来。否则容易冲突。

**putVal()**
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        // resize()是扩容方法，下面再讲。
        n = (tab = resize()).length;
    // key位置为null，直接加入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 如果key相同，则覆盖
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 赋值在后面
            e = p;
        // 如果是红黑树，执行红黑树的putTreeVal()
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 否则是链表
        else {
            for (int binCount = 0; ; ++binCount) {
                // 到达链表的尾部
                if ((e = p.next) == null) {
                    // 在尾部插入新节点
                    p.next = newNode(hash, key, value, null);
                    // 达到红黑树的阈值，转换
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果发现key相同，跳出
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // key相同，应当覆盖
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // putVal()传过来的onlyIfAbsent=false
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 回调函数，LinkedHashMap中有实现
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 超出阈值扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);// 回调函数，LinkedHashMap中有实现
    return null;
}
```
**resize()**

```java
final Node<K,V>[] resize() {
        // table是存储元素的数组，总是2的幂次倍
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            /**
             超过最大容量不扩容
             MAXIMUM_CAPACITY = 1 << 30
             这里我疑惑为啥不是直接用int最大值
             不过想起来HaspMap要求数组必须是2的幂
             因为int最大是2^31-1,可惜不符合要求。
             */
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        /**进入到这说明oldCap = 0 而且 threshold > 0
        那么它只有通过这几种构造方法才会来这：
        1.HashMap(int initialCapacity, float loadFactor)
        2.HashMap(int initialCapacity) ，传入容量0
        3.HashMap(Map<? extends K, ? extends V> m)导致oldTab为null
        但是其中都对threshold做了设置
        threshold = tableSizeFor(initialCapacity)
        */
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        // 如果 threshold 也为0 了
        // 说明他是调用HashMap()
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 新阈值为0，不过我看不出来怎么还能为0，可能以防万一吧。
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        // 抑制编译器警告
        @SuppressWarnings({"rawtypes","unchecked"})   
            // 初始化table
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        // copy数据
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 如果此节点没有next节点，直接放入新数组中
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // 如果是红黑树，则进行相关操作
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    // 否则就是链表
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 索引不变
                            if ((e.hash & oldCap) == 0) {
                                // 第一个设为头节点
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 索引改变
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 原table位置中有数据
                        // 此时loHead肯定不为null
                        if (loTail != null) {
                            loTail.next = null;
                            // 头节点直接放进去
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // 所不同的是索引变了
                            // 为何是+oldCap？详见下面
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
由于我们使用的是2的幂，所以元素要么是原位置，要么是原位置加上原数组长度。  

```java
n-1 0000 1111  
5   0000 0101  5&(n-1)  : 0000 0101   
213 1101 0101  213&(n-1)：0000 0101  

扩容后 n = 32
n-1 0001 1111
5   0000 0101  5&(n-1)  : 0000 0101 
213 1101 0101  213&(n-1)：0001 0101
```
n=16时，只有低四位参与了运算。
n=32时，低5位参与了运算，所以对于之前的数来说。此位有可能为0，也有可能为1。  
所以上面对此bit进行`与`运算

```java
oldCap 0001 0000
(e.hash & oldCap) == 0 则说明索引不变
0000 0101 & oldCap -> 0000 0000

1101 0101 & oldCap -> 0001 0000
```

### 常见问题
**为什么HashMap线程不安全？**  
首先由于HashMap没有做任何并发处理，导致多线程情况下，操作同一个Bucket，容易出现问题，具体就不说了。
其次 在jdk1.8之前，`resize()`方法采用的是头插入,容易造成死链。  
详见：[Java HashMap的死循环](https://coolshell.cn/articles/9606.html)

**HashMap与HashTable区别**  
1. HashTable是线程安全的，方法经过`synchronized`修饰,也因如此，性能较低。多线程应该使用ConcurrentHashMap。
2. HashMap允许键值为null，不过键只能有一个null；而HashTable则会抛出NPE。
3. 初始容量不同，扩容机制不同。如果不指定大小，HashTable默认是11，每次扩充是2n+1。HashMap默认是16，每次扩充是原来2倍。如果指定大小，HashTable直接使用给定大小，而HashMap则使用`tableSizeFor()`确定大小。
4. HashMap底层使用链表+红黑树，而HashTable没有，估计没什么人用，没人管了。
 

**HashMap 的⻓度为什么是2的幂次⽅**  
经过上面的分析我们得知HashMap中确定索引位置是`(n-1)&hash`。我们想把一个数放入数组中，确定其位置最简单的做法就是与数组长度取余 `hash%n`。因为这里n是2的幂，所以`(n-1)&hash` **等同于**`hash%n` 。位运算比%效率高，很多地方也是用到了位运算。

**HashMap 的最大⻓度为什么是2^30**  
因为HashMap规定⻓度必须是2的幂，但是int最大值是2^31 -1,所以能找到最大的数就只有了2^30了。

**HashMap在jdk1.7与1.8的区别**  
[Hashmap的结构，1.7和1.8有哪些区别](https://blog.csdn.net/qq_36520235/article/details/82417949)

参考

[一文读懂HashMap](https://www.jianshu.com/p/ee0de4c99f87)  
[集合框架源码学习之HashMap(JDK1.8)](https://juejin.im/post/5ab0568b5188255580020e56)  
[HashMap方法hash()、tableSizeFor()](https://blog.csdn.net/huzhigenlaohu/article/details/51802457)






---




## 1.1.4 ConcurrentHashMap

### 简介
ConcurrentHashMap从JDK1.5开始随java.util.concurrent包一起引入JDK中。

引用一段《Java并发编程的艺术》的话（大致意思），
需要注意的是**此书不是基于jdk1.8写的**。

**HashMap** ：先说HashMap，HashMap是线程不安全的，在并发环境下，可能会形成环状链表（jdk1.7扩容时可能造成，具体原因自行百度google或查看源码分析），导致get操作时，cpu空转，所以，在并发环境中使用HashMap是非常危险的。

**HashTable** ： HashTable和HashMap的实现原理几乎一样，差别无非是1.HashTable不允许key和value为null；2.HashTable是线程安全的。但是HashTable线程安全的策略实现代价却太大了，简单粗暴，get/put所有相关操作都是synchronized的，这相当于给整个哈希表加了一把大锁，多线程访问时候，只要有一个线程访问或操作该对象，那其他线程只能阻塞，相当于将所有的操作**串行化**，在竞争激烈的并发场景中性能就会非常差。
![HashTable](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/1024555-20170514173954488-1353945142.png)

**ConcurrentHashMap**：
HashTable性能差主要是由于所有操作需要竞争同一把锁，而如果容器中有多把锁，每一把锁锁一段数据，这样在多线程访问时不同段的数据时，就不会存在锁竞争了，这样便可以有效地提高并发效率。这就是ConcurrentHashMap所采用的"**分段锁**"思想。
![ConcurrentHashMap分段锁](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/1024555-20170514174100832-1891630860.png)

jdk1.8中不是使用分段锁了，而是**CAS**和**synchronized**。下面就从jdk1.7和1.8源码分析一下。

### JDK1.7

参考[ConcurrentHashMap实现原理及源码分析](https://www.cnblogs.com/chengxiao/p/6842045.html)吧。

只说几个重要方法（以下文字来自《Java并发编程的艺术》）。

#### get操作 
get操作的高效之处在于**整个get过程不需要加锁**，除非读到的值是空才会加锁重读。我们知道HashTable容器的get方法是需要加锁的，那么ConcurrentHashMap的get操作是如何做到不加锁的呢？原因是它的get方法里将要使用的共享变量都定义成volatile，如用于统计当前Segement大小的count字段和用于存储值的HashEntry的value。定义成volatile的变量，能够在线程之间保持可见性，能够被多线程同时读，并且保证不会读到过期的值，但是只能被单线程写（有一种情况可以被多线程写，就是写入的值不依赖于原值），在get操作里只需要读不需要写共享变量count和value，所以可以不用加锁。之所以不会读到过期的值，是根据java内存模型的happen before原则，对volatile字段的写入操作先于读操作，即使两个线程同时修改和获取volatile变量，get操作也能拿到最新的值，这是用volatile替换锁的经典应用场景。

#### put操作  
由于put方法里需要对共享变量进行写入操作，所以为了线程安全，在操作共享变量时必须得加锁。Put方法首先定位到Segment，然后在Segment里进行插入操作。插入操作需要经历两个步骤，第一步判断是否需要对Segment里的HashEntry数组进行扩容，第二步定位添加元素的位置然后放在HashEntry数组里。

**是否需要扩容**  
在插入元素前会先判断Segment里的HashEntry数组是否超过容量（threshold），如果超过阀值，数组进行扩容。值得一提的是，Segment的扩容判断比HashMap更恰当，因为HashMap是在插入元素后判断元素是否已经到达容量的，如果到达了就进行扩容，但是很有可能扩容之后没有新元素插入，这时HashMap就进行了一次无效的扩容。

**如何扩容**  扩容的时候首先会创建一个两倍于原容量的数组，然后将原数组里的元素进行再hash后插入到新的数组里。为了高效ConcurrentHashMap不会对整个容器进行扩容，而只对某个segment进行扩容。

#### size操作
如果我们要统计整个ConcurrentHashMap里元素的大小，就必须统计所有Segment里元素的大小后求和。Segment里的全局变量`count`是一个**volatile**变量，那么在多线程场景下，我们是不是直接把所有Segment的`count`相加就可以得到整个ConcurrentHashMap大小了呢？不是的，虽然相加时可以获取每个Segment的`count`的最新值，但是拿到之后可能累加前使用的`count`发生了变化，那么统计结果就不准了。所以最安全的做法，是在统计size的时候把所有Segment的`put`，`remove`和`clean`方法全部锁住，但是这种做法显然非常低效。

因为在累加count操作过程中，之前累加过的count发生变化的几率非常小，所以ConcurrentHashMap的做法是先**尝试2次通过不锁住Segment**的方式来统计各个Segment大小，如果统计的过程中，容器的count发生了变化，则再**采用加锁**的方式来统计所有Segment的大小。

那么ConcurrentHashMap是如何判断在统计的时候容器是否发生了变化呢？使用modCount变量，在put , remove和clean方法里操作元素前都会将变量modCount进行加1，那么在统计size前后比较modCount是否发生变化，从而得知容器的大小是否发生变化。


### JDK1.8
ConcurrentHashMap底层与HashMap差不多，所以很多属性也有，这里就介绍下不同的地方。
#### 常量

```java
// stride为每个cpu所需要处理的桶个数
// 最小转移步长：由于在扩容过程中，会把一个待转移的数组分为多个区间段（转移步长），每个线程一次转移一个区间段的数据，
// 一个区间段（转移步长）的默认长度是16，实际运行过程中会动态计算
private static final int MIN_TRANSFER_STRIDE = 16;
/**扩容戳有效位数：每次在需要扩容的时会根据当前数组table的大小生成一个扩容戳，
 * 当一个线程需要协助扩容时需要实时计算扩容戳来验证是否 
 * 需要协助扩容或扩容过程是否完成，
 * 生成扩容戳的方式：Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
 * 其中n表示当前table的大小，利用该常量表示扩容戳的有效位长度，默认16位。 
*/
private static int RESIZE_STAMP_BITS = 16;
// 2^15-1，最大并发扩容线程数：65535
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
// 32-16=16，sizeCtl中记录size大小的偏移量
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
// forwarding nodes的hash值
static final int MOVED     = -1;
// 树根节点的hash值
static final int TREEBIN   = -2;
// ReservationNode的hash值
static final int RESERVED  = -3;
// 可用处理器数量
static final int NCPU = Runtime.getRuntime().availableProcessors();
//存放node的数组
transient volatile Node<K,V>[] table;
/**控制标识符，用来控制table的初始化和扩容的操作，不同的值有不同的含义
 *当为负数时：-1代表正在初始化，-N代表有N-1个线程正在 进行扩容
 *当为0时：代表当时的table还没有被初始化
 *当为正数时：表示初始化或者下一次进行扩容的大小*/
private transient volatile int sizeCtl;
```
#### 属性

```java
// 代表Map中元素个数的的基础计数器，当无竞争时直接使用CAS方式更新该数值 
transient volatile long baseCount; 
/** 
* sizeCtl用于table初始化和扩容控制，当为正数时表示table的扩容阈值（n * 0.75），当为负数时表示table正在初始化或者扩容，
* -1表示table正在初始化，其他负值代表正在扩容，第一个扩容的线程会把扩容戳rs左移RESIZE_STAMP_SHIFT(默认16)位再加2更新设置到sizeCtl中(sizeCtl = (rs << 16) + 2)，
* 每次一个新线程来扩容时都令sizeCtl = sizeCtl + 1，因此可根据sizeCtl计算出正在扩容的线程数，注释中所
* 描述的 sizeCtl = -(1+threads)是不准确的。扩容时sizeCtl有两部分组成，第一部分是扩容戳，占据sizeCtl的高有效位，长度为
* RESIZE_STAMP_BITS位（默认16），剩下的低有效位长度为32-RESIZE_STAMP_BITS位（16），每个新线程协助扩容时sizeCtl+1
* ，直到所有的低有效位被占满，低有效位默认占16位（最高位为符号位），所以扩容线程数默认最大为65535
*/
transient volatile int sizeCtl; 
/** 
* 用于控制多个线程去扩容时领取扩容子任务，每个线程领取子任务时，要减去扩容步长，如果能减成功，
* 则成功领取一个扩容子任务，`transferIndex = transferIndex - stride(扩容步长)`，transferIndex减到0时
* 代表没有可以领取的扩容子任务。
*/
transient volatile int transferIndex; 
// 扩容或者创建CounterCells时使用的自旋锁（使用CAS实现）；
transient volatile int cellsBusy; 
/** 
* 存储Map中元素的计数器，当并发量较高时`baseCount`竞争较为激烈，更新效率较低，所以把变化的数值
* 更新到`counterCells`中的某个节点上，计算size()时需要统计每个`counterCells`的大小再加上`baseCount`的数值。
*/
transient volatile CounterCell[] counterCells;
/**
* ConcurrentHashMap采用cas算法进行更新变量（table[i]，sizeCtl，transferIndex，cellsBusy等）来保证线程安全性，它其实是一种乐观策略，
* 假设每次都不会产生冲突，所以能够直接更新成功，如果出现冲突则再重试，直到更新成功。实现cas主要是借助了`sun.misc.Unsafe`类，该类提供了
* 诸多根据内存偏移量直接从内存中读取设置对象属性的底层操作。
*/
static final sun.misc.Unsafe U;

```


#### 重要方法
**Unsafe提供的方法**
```java
// 获取table中index为i的节点
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
// table中index为i的节点更新为v（cas原子更新方式）
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
// 把table中index为i的节点更新为v（普通更新方式）
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```


**putVal()**

和HashMap一样，`put`方法调用的是内部的`putVal()`
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 此hash算法与HashMap一样
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 没有节点，cas插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 如果要插入的位置的节点是转移节点，
        // 说明Map正在扩容，则协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 对此节点加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                // hashCode>0,代表节点为单链表
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                //key在Map中已经存在，更新key的值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                // 如果没有重复的，插入到链表末端
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                // 否则是红黑树
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
    //更新Map的节点计数
    addCount(1L, binCount);
    return null;
}
```
**put流程**
1. 首先是根据key的hashCode做一次重哈希（进一步减少哈希碰撞），与HashMap一样的算法。
2. 先判断table为空，空则初始化Map，否则：
3. 根据hashCode取模定位到table中的某个节点f，如果f为空，则新创建一个节点，使用cas方式更新到数组的f节点上，插入结束，否则：
4. 若f是转移节点，则调用`helpTransfer`协助转移，否则：
5. 锁定节点f（通过synchronized加锁）
6. 节点f锁定成功后判断节点f类型，如果f是链表节点，则直接插入到链表底端（key不存在的话），如果节点f是红黑树节点，则按照二叉搜索树的方式插入节点，并调整树结构使其满足红黑规则
7. 最后调用`addCount`更新Map中的节点计数。


**addCount()**

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //更新baseCount，table的数量，counterCells表示元素个数的变化
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        //如果多个线程都在执行，则CAS失败，执行fullAddCount，全部加入count
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    // 表示需要进行检查扩容操作
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
// map中的元素个数已经大于扩容阈值且小于最大阈值，table需要扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 计算扩容戳
            int rs = resizeStamp(n);
            // 表示正在扩容或者table初始化
            if (sc < 0) {
// sizeCtl无符号右移16位得到扩容戳，扩容戳不同说明当前线程已经滞后其他线程，
//其他线程已经开启了新一轮扩容任务，不能再去扩容。
//或者 扩容线程数大于最大扩容线程数，nextTable为空表示没有在扩容，不需要协助
//或者transferIndex < 0 表示其他线程已经把扩容子任务领取完毕，也不需要协助扩容
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
// 使用cas方式把sizeCtl加1，代表增加一个协助扩容的线程，
//并令当前线程去协助扩容，当前线程协助完成后需要把sizeCtl减1，
//所以sizeCtl<0时可以利用sizeCtl计算出扩容线程的个数
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
//当前没有线程在扩容，则把扩容戳rs 左移16位加2得到一个负值，
//用cas方式更新到sizeCtl中，更新成功则作为第一个扩容线程执行扩容任务
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

**helpTransfer**

helpTransfer 主要用于通过计算来验证Map是否需要协助扩容，如果Map正在扩容且扩容未结束则协助扩容，并通过transfer执行扩容过程。

```java
/**
 * Helps transfer if a resize is in progress.
 * 当其他线程进来时发现当前Map正在扩容，则判断是否需要帮助扩容
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
//当前table数组和nextTable数组都不为空，
//而且当前节点为ForwardingNode，说明HashMap正在扩容
    if (tab != null && (f instanceof ForwardingNode) &&              
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
//以当前数组的大小通过移位的方式生成扩容戳，保证每次扩容过程都生成唯一的扩容戳
        int rs = resizeStamp(tab.length); 
//指向table的指针和nextTab的指针没有被其他线程更新
// sizeCtl小于0时，可以通过移位反解出正在扩容的线程数，
//代表正在扩容，大于0时代表下次扩容的阈值
        while (nextTab == nextTable && table == tab && 
               (sc = sizeCtl) < 0) {
//sc右移16位若等于rs，说明没有线程在扩容
//扩容线程数超过限制
//transferIndex用于每次分配数据转移任务的上界，如果小于0则说明没有可以分配的数据转移任务
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs    
                   || sc == rs + 1 || sc == rs + MAX_RESIZERS  
                   || transferIndex <= 0)         
                break;
//cas方式更新sc，更新成功则扩容线程数+1，并开始帮助转移任务
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```
**transfer**

transfer方法为实际的扩容实现，实现过程有些复杂。

```java
/**
     * Moves and/or copies the nodes in each bin to new table. See
     * above for explanation.
     */
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        //根据table的长度及cpu核数计算转移任务步长
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE) // 计算转移步长并判断是否小于最小转移步长
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 第一个扩容线程进来需要初始化nextTable
        if (nextTab == null) {            
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true; 
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
//当前索引已经走到了本次扩容子任务的下界，子任务转移结束
                if (--i >= bound || finishing) 
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {//任务转移完成
                    i = -1;
                    advance = false;
                }
               //通过cas方式尝试获取一个转移任务（transferIndex - 转移步长stride），获取成功后得到处理的下界及当前索引
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;  // 更新当前子任务的下界
                    i = nextIndex - 1;  // 更新当前index位置  
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) { // 扩容结束
                int sc;
// 最后一个出去的线程：更新table指针及sizeCtl值
                if (finishing) {             
                    nextTable = null;
                    table = nextTab;     // 指向扩容后的数组
//sizeCtl更新为最新的扩容阈值（2n - 0.5n = 1.5n =  2n * 0.75），移位实现保证高效率
                    sizeCtl = (n << 1) - (n >>> 1);  
                    return;
                }
                // sizeCtl减1，表示减少一个扩容线程
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
// 判断是否是最后一个扩容线程，如果不是则直接退出，由于第一个线程进来时把扩容戳rs左移16位+2更新到sizeCtl，
//所以如果是最后一个线程的话，sizeCtl -2 应该等于rs左移16位
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
//如果是最后一个线程，则结束标志更新为真，并且在重新检查一遍数组
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
// 当前桶节点为空，设置为转移成转移节点
            else if ((f = tabAt(tab, i)) == null)    
                advance = casTabAt(tab, i, null, fwd);
            // 该桶节点已经被转移
            else if ((fh = f.hash) == MOVED)   
                advance = true; // already processed
            else {
                synchronized (f) {         // 获取该节点的锁
// 获取锁之后再次验证是否被其他线程修改过
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
// 节点HasCode大于0 代表该节点为链表节点
                        if (fh >= 0) {        
// 由于数组长度n为2的幂次方，所以当数组长度增加到2n时，
//原来hash到table中i的数据节点在长度为2n的table中要么在低位nextTab[i]处，
//要么在高位nextTab[n+i]处，具体在哪个位置与(fh & n)的计算结果有关
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
// 此处循环的目的是找到链表中最后一个从低索引位置变
//到高索引位置或者从高索引位置变到低索引位置的节点lastRun
//从lastRun节点到链表的尾节点可根据runBit直接插入到
//新数组nextTable的节点中，其目的是尽量减少新创建节点数量，直接更新指针位置
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
// 对于lastRun之前的链表节点，根据hashCode&n可确定即将转移到nextTable中的低索引位置节点（nextTab[i]）
//还是高索引位置节点（nextTab[i + n]），并形成两个新的链表
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
// 使用cas方式更新两个链表到新数组nextTable中，并且把原来的table节点i中的数值变为转移节点
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
//该节点为二叉搜索数节点（红黑树）
                        else if (f instanceof TreeBin) {   
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }

```
`transfer`数据转移过程可以分为如下几个步骤：
1. 第一个扩容线程进来后创建nextTable数组，并设置transferIndex；
2. 线程（第一个或其他）通过transferIndex-stride（扩容步长）来领取一个扩容子任务，transferIndex减到0说明所有子任务领取完成；
3. 线程领取到扩容子任务后设置当前处理子任务的下界并更新当前处理节点所在的索引位置；
4. 对子任务中的每个节点，扩容线程从后向前依次判断该节点是否已经转移，如果没有转移，则对该节点进行加锁，并且把节点对应的链表或红黑树转移到新数组nextTable中去；
5. 如果线程处理的节点索引已经到达子任务的下界，则子任务执行结束，并尝试去领取新的子任务，若领取不到再判断当前线程是否是最后一个扩容线程，若是则最后扫描一遍数组，执行清理工作，否则直接退出。


参考 

[ConcurrentHashMap之扩容实现（基于JDK1.8)](https://www.jianshu.com/p/fc72281e529f)

[ConcurrentHashMap 1.7和1.8区别](https://blog.csdn.net/xingxiupaioxue/article/details/88062163)