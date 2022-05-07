
# List源码

## 1.1 ArrayList

### 1.1.1 ArrayList属性

```java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, Serializable {
	// 序列化id
	private static final long serialVersionUID = 8683452581122892189L;
	// 默认初始的容量
	private static final int DEFAULT_CAPACITY = 10;
	// 一个空对象
	private static final Object[] EMPTY_ELEMENTDATA = new Object[0];
	// 一个空对象，如果使用默认构造函数创建，则默认对象内容默认是该值
	private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = new Object[0];
	// 当前数据对象存放地方，当前对象不参与序列化
	transient Object[] elementData;
	// 当前数组长度
	private int size;
	// 数组最大长度
	private static final int MAX_ARRAY_SIZE = 2147483639;
 
	// 省略方法。。
}
```

### 1.1.2 构造函数

无参构造
```java
 public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

指定长度构造
```java
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
```

collection对象转换成数组
```java
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

### 1.1.3 add方法

#### 1.1.3.1 默认添加到末尾

```java
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

#### 1.1.3.2 添加到指定位置

```java
public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

#### 1.1.3.3 默认添加集合到末尾

```java
public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
```

#### 1.1.3.4 集合添加到指定位置

```java
public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
```

### 1.1.4 get方法

```java
 public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```

### 1.1.5 set方法

```java
 public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```

### 1.1.6 remove方法

#### 1.1.6.1 删除指定位置

```java
public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

#### 1.1.6.2 删除指定元素

```java
public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
```

#### 1.1.6.3 删除集合中包含的元素

```java
public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }
```

#### 1.1.6.3 删除指定区间的元素

```java
protected void removeRange(int fromIndex, int toIndex) {
        modCount++;
        int numMoved = size - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }
```

### 1.1.7 iterator方法

interator方法返回的是一个内部类，由于内部类的创建默认含有外部的this指针，所以这个内部类可以调用到外部类的属性

调用完iterator之后，我们会使用iterator做遍历，这里使用next做遍历的时候有个需要注意的地方，就是调用next的时候，可能会引发ConcurrentModificationException，当修改次数，与期望的修改次数（调用iterator方法时候的修改次数）不一致的时候会发生异常

```java
public Iterator<E> iterator() {
    return new Itr();
}

@SuppressWarnings("unchecked")
public E next() {
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}
```

### 1.1.8 总结

#### 1.1.8.1 Arrays.copyOf

- original - 要复制的数组 
- newLength - 要返回的副本的长度 
- newType - 要返回的副本的类型

其实Arrays.copyOf底层也是调用System.arraycopy实现的源码如下：
```java
//基本数据类型（其他类似byte，short···）
public static int[] copyOf(int[] original, int newLength) {
        int[] copy = new int[newLength];
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```


#### 1.1.8.2 System.arraycopy

- src	原数组
- srcPos	原数组
- dest	目标数组
- destPos	目标数组的起始位置
- length	要复制的数组元素的数目

该方法被标记了native，调用了系统的C/C++代码，在JDK中是看不到的，但在openJDK中可以看到其源码。该函数实际上最终调用了C语言的`memmove()`函数，因此它可以保证同一个数组内元素的正确复制和移动，比一般的复制方法的实现效率要高很多，很适合用来批量处理数组。Java强烈推荐在复制大量数组元素时用该方法，以取得更高的效率




## 1.2 LinkedList

### 1.2.1 LinkedList属性

相比于ArrayList，LinkedList额外实现了双端队列接口Deque，这个接口主要是声明了队头，队尾的一系列方法

```java
	// LinkedList节点个数
    transient int size = 0;

    /**
     * Pointer to first node. 指向头结点
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.  指向尾节点
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;

    ...

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

### 1.2.2 构造函数

```java
public LinkedList() {
    //空参构造函数
}

public LinkedList(Collection<? extends E> c) {
        this();
        // 将集合添加到链表中去
        addAll(c);
    }

public boolean addAll(Collection<? extends E> c) {
        // 从链表尾巴开始集合中元素
        return addAll(size, c);
    }

public boolean addAll(int index, Collection<? extends E> c) {
    // 1.添加位置的下标的合理性检查
    checkPositionIndex(index);

    // 2.将集合转换为Object[]数组对象
    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;

    // 3.得到插入位置的前继节点和后继节点
    Node<E> pred, succ;
    if (index == size) {
        // 从尾部添加的情况：前继节点是原来的last节点；后继节点是null
        succ = null;
        pred = last;
    } else {
        // 从指定位置（非尾部）添加的情况:前继节点就是index位置的节点，后继节点是index位置的节点的前一个节点
        succ = node(index);
        pred = succ.prev;
    }
    
    // 4.遍历数据，将数据插入
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        // 创建节点
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            // 空链表插入情况：
            first = newNode;
        else
            // 非空链表插入情况：
            pred.next = newNode;
        // 更新前置节点为最新插入的节点（的地址）
        pred = newNode;
    }

    if (succ == null) {
        // 如果是从尾部开始插入的，则把last置为最后一个插入的元素
        last = pred;
    } else {
        // 如果不是从尾部插入的，则把尾部的数据和之前的节点连起来
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;  // 链表大小+num
    modCount++;  // 修改次数加1
    return true;
}

```

### 1.2.3 add造函数

#### 1.2.3.1 add(E e)

```java
// 作用：将元素添加到链表尾部
public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    final Node<E> l = last;  // 获取尾部元素
    final Node<E> newNode = new Node<>(l, e, null); // 以尾部元素为前继节点创建一个新节点
    last = newNode;  // 更新尾部节点为需要插入的节点
    if (l == null)
        // 如果空链表的情况：同时更新first节点也为需要插入的节点。（也就是说：该节点既是头节点first也是尾节点last）
        first = newNode;
    else
        // 不是空链表的情况：将原来的尾部节点（现在是倒数第二个节点）的next指向需要插入的节点
        l.next = newNode;
    size++; // 更新链表大小和修改次数，插入完毕
    modCount++;
}
```

#### 1.2.3.2  add(int index, E element)

```java
// 作用：在指定位置添加元素
public void add(int index, E element) {
    // 检查插入位置的索引的合理性
    checkPositionIndex(index);

    if (index == size)
        // 插入的情况是尾部插入的情况：调用linkLast（）解释如上。
        linkLast(element);
    else
        // 插入的情况是非尾部插入的情况（中间插入）：linkBefore（）见下面。
        linkBefore(element, node(index));
}

private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}

void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;  // 得到插入位置元素的前继节点
    final Node<E> newNode = new Node<>(pred, e, succ);  // 创建新节点，其前继节点是succ的前节点，后接点是succ节点
    succ.prev = newNode;  // 更新插入位置（succ）的前置节点为新节点
    if (pred == null)
        // 如果pred为null，说明该节点插入在头节点之前，要重置first头节点 
        first = newNode;
    else
        // 如果pred不为null，那么直接将pred的后继指针指向newNode即可
        pred.next = newNode;
    size++;
    modCount++;
}
```

### 1.2.4 get(int index)

```java
public E get(int index) {
    // 元素下表的合理性检查
    checkElementIndex(index);
    // node(index)真正查询匹配元素并返回
    return node(index).item;
}

// 作用：查询指定位置元素并返回
Node<E> node(int index) {
    // assert isElementIndex(index);

    // 如果索引位置靠链表前半部分，从头开始遍历
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
    // 如果索引位置靠链表后半部分，从尾开始遍历
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
```

### 1.2.5 remove(int index)

```java
// 作用：移除指定位置的元素
public E remove(int index) {
    // 移除元素索引的合理性检查
    checkElementIndex(index);
    // 将节点删除
    return unlink(node(index));
}

    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;  // 得到指定节点的值
        final Node<E> next = x.next; // 得到指定节点的后继节点
        final Node<E> prev = x.prev; // 得到指定节点的前继节点

        // 如果prev为null表示删除是头节点，否则就不是头节点
        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null; // 置空需删除的指定节点的前置节点（null）
        }

        // 如果next为null,则表示删除的是尾部节点，否则就不是尾部节点
        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null; //  置空需删除的指定节点的后置节点
        }
        
        // 置空需删除的指定节点的值
        x.item = null;
        size--; // 数量减1
        modCount++;
        return element;
    }
```

### 1.2.6 clear

```java
// 清空链表
public void clear() {
    // 进行for循环，进行逐条置空；直到最后一个元素
    for (Node<E> x = first; x != null; ) {
        Node<E> next = x.next;
        x.item = null;
        x.next = null;
        x.prev = null;
        x = next;
    }
    // 置空头和尾为null
    first = last = null;
    size = 0;
    modCount++;
}
```

### 1.2.7 indexOf(Object o)

```java
 // 返回元素在链表中的索引，如果不存在则返回-1
  public int indexOf(Object o) {
        int index = 0;
        // 如果元素为null，进行如下循环判断
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
        // 元素不为null.进行如下循环判断
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
```

### 1.2.8 总结

1. ArrayList是实现了基于动态数组的数据结构，而LinkedList是基于链表的数据结构；
2. 对于随机访问get和set，ArrayList要优于LinkedList，因为LinkedList要移动指针；
3. 对于添加和删除操作add和remove，一般大家都会说LinkedList要比ArrayList快，因为ArrayList要移动数据。但是实际情况并非这样，对于添加或删除，LinkedList和ArrayList并不能明确说明谁快谁慢,因为它是双向链表，所以在第2个元素后面插入一个数据和在倒数第2个元素后面插入一个元素在效率上基本没有差别，但是ArrayList由于要批量copy的元素越来越少，操作速度必然追上乃至超过LinkedList





## 1.3 CopyOnWriteArrayList

Redis4之后支持RDB-AOF混合持久化的方式

bgsave操作使用CopyOnWrite机制进行写时复制，是由一个子进程将内存中的最新数据遍历写入临时文件，此时父进程仍旧处理客户端的操作，当子进程操作完毕后再将该临时文件重命名为dump.rdb替换掉原来的dump.rdb文件，因此无论bgsave是否成功，dump.rdb都不会受到影响



### 1.3.1 CopyOnWrite容器实现原理

CopyOnWrite容器即写时复制的容器，是一种用于程序设计中的优化策略，同样属性的还有`CopyOnWriteArraySet`

写时复制（读写分离的思想）：读取时不加锁只是写入和删除时加锁实现机制（volatile + ReentrantLock）

往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器

CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。【当执行add或remove操作没完成时，get获取的仍然是旧数组的元素】


### 1.3.2 初始化

```java
public CopyOnWriteArrayList() {
    setArray(new Object[0]);    //创建一个大小为0的Object数组作为array初始值
}
public CopyOnWriteArrayList(E[] toCopyIn) {
    //创建一个list，其内部元素是toCopyIn的的副本
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
//将传入参数集合中的元素复制到本list中
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
```

### 1.3.2 添加元素

`add(E e)` 往集合末尾添加一个元素

```java
public boolean add(E e) {
    // 使用juc的可重入锁保证线程安全
    // 其中lock在CopyOnWriteArrayList的定义：final transient ReentrantLock lock = new ReentrantLock();
    final ReentrantLock lock = this.lock;
    // 线程得到锁，同时只能有一个线程操作
    lock.lock();
    try {
        // 获取当前的数组，和ArrayList一样。底层就是一个数组。
        Object[] elements = getArray();
        // 得到数组的长度
        int len = elements.length;
        // 直接copy出一个新数组，传入的参数是旧数组和新数组的容量。返回一个新数组，新数组数据就是就数组的数组，并且新数组的容量+1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 往elements.length添加一个新元素。也就是末尾添加一个元素。
        newElements[len] = e;
        // 用新数组覆盖旧数组
        setArray(newElements);
        // 添加成功，返回true
        return true;
    } finally {
        // 当前线程操作完成，释放锁。
        lock.unlock();
    }
}
```

`add(int index, E element)` 往集合指定位置添加一个元素
```java
public void add(int index, E element) {
    // 上面add(E e)一样的逻辑就不再赘述
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 判断插入的位置是否合法。
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+", Size: "+len);
        Object[] newElements;
        int numMoved = len - index;
        // 如果插入的是数组的最后一个位置，直接copy一个新数组，并且新数组的容量+1
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            // 如果插入的不是最后一个位置，则new一个新数组，新数组的容量在原来的旧数组容量上+1
            newElements = new Object[len + 1];
            // 把旧数组直接拷贝到空的新数组中。
            // System.arraycopy参数解释：
            // 第一个参数是需要被拷贝的旧数组，第二个参数从旧数组的什么位置开始拷贝，
            // 第三个参数是拷贝到的目标数组，第四个参数是从旧数组的第几个参数开始拷贝，
            // 第五个参数是从旧数组拷贝多少个元素到新数组。
            // 第一次拷贝：把旧数组拷贝到新数组，把要插入的位置的index之前的数据拷贝到新数组。
            System.arraycopy(elements, 0, newElements, 0, index);
            // 第二次拷贝：把旧数组从要插入的位置开始拷贝到新数组要插入的位置+1开始，把剩下的旧数组数据拷贝到新数组。
            // 这里index+1就在新数组中预留了一个插入位置。
            System.arraycopy(elements, index, newElements, index + 1, numMoved);
        }
        // 往index的位置插入元素
        newElements[index] = element;
        // 覆盖旧数组
        setArray(newElements);
    } finally {
        // 线程执行完毕，解锁
        lock.unlock();
    }
}
```

### 1.3.3 修改元素

```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();    //加锁
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);  //先得到要修改的旧值

        if (oldValue != element) {  //值确实修改了
            int len = elements.length;
            //将array复制到新数组，并进行修改，并设置array为新数组
            Object[] newElements = Arrays.copyOf(elements, len);			
            newElements[index] = element;
            setArray(newElements);
        } else {
            // 虽然值确实没改，但要保证volatile语义，需重新设置array
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

### 1.3.4 删除元素

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);    //得到要删除的元素
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                                numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

### 1.3.5 迭代器的弱一致性

弱一致性是指返回迭代器后，其它线程对list的增删改对迭代器是不可见的，迭代器返回对象是快照

```java
public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);
}
static final class COWIterator<E> implements ListIterator<E> {
        //array的快照
        private final Object[] snapshot;
        //数组下标
        private int cursor;
        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
        }
        public boolean hasNext() {
            return cursor < snapshot.length;
        }
        public boolean hasPrevious() {
            return cursor > 0;
        }
        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }
}
```