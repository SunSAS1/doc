# 1.1 Java集合

> 作者：SunSAS
>
> **介绍：** SunSAS是SunSAS

## 1.1.1 ArrayList

### ArrayList简介
ArrayList 的底层是数组队列，相当于动态数组。与 Java 中的数组相比，它的容量能动态增长。在添加大量元素前，应用程序可以使用`ensureCapacity`操作来增加 ArrayList 实例的容量。这可以减少递增式再分配的数量。

![ArrayList类结构](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/QQ%E5%9B%BE%E7%89%8720200709151618.png)

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

![LinkedList类结构图](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/QQ%E5%9B%BE%E7%89%8720200710112156.png)

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

