
# Map源码

## 1.1 HashMap源码1.7

### 1.1.1 HashMap的属性
```java
//默认大小
static final int DEFAULT_INITIAL_CAPACITY = 16;
//最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;
//默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//存储元素的数组
transient Entry[] table;
//键值对数量
transient int size;
//阈值
int threshold;
//负载因子
final float loadFactor;
//修改次数 快速失败
transient volatile int modCount;
```

table是存储Hashmap键值对，Entry是一个内部类，存储着键值对、hash值以及下一个节点地址

```java
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    final int hash;
    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }
```

### 1.1.2 核心api

```java
V get(Object key); // 获得指定键的值
V put(K key, V value);  // 添加键值对
void putAll(Map<? extends K, ? extends V> m);  // 将指定Map中的键值对 复制到 此Map中
V remove(Object key);  // 删除该键值对

boolean containsKey(Object key); // 判断是否存在该键的键值对；是 则返回true
boolean containsValue(Object value);  // 判断是否存在该值的键值对；是 则返回true
 
Set<K> keySet();  // 单独抽取key序列，将所有key生成一个Set
Collection<V> values();  // 单独value序列，将所有value生成一个Collection

void clear(); // 清除哈希表中的所有键值对
int size();  // 返回哈希表中所有 键值对的数量 = 数组中的键值对 + 链表中的键值对
boolean isEmpty(); // 判断HashMap是否为空；size == 0时 表示为 空 

```

### 1.1.3 构造方法

无参构造
```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    threshold = (int)(DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR);
    table = new Entry[DEFAULT_INITIAL_CAPACITY];
    init();
}
```

有参构造
```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);

    // Find a power of 2 >= initialCapacity
    int capacity = 1;
    while (capacity < initialCapacity)
        capacity <<= 1;

    this.loadFactor = loadFactor;//初始化加载因子
    threshold = (int)(capacity * loadFactor); 
    table = new Entry[capacity];
    init();//供子类扩展的方法
}
```

### 1.1.4 hashcode的取值算法

`hashmap为什么容量是2的幂次？`

因为2的幂-1都是11111结尾的，所以碰撞几率小

保证只有一个byte位置上面是1 这个在length-1的时候得到一个有规律得数字，因为`只有一个byte上是1的数字才是2的幂次方`

没有使用%计算也是因为jdk中取余操作性能比较慢相比按位操作

```java
static int indexFor(int h, int length) {
        return h & (length-1);
    }
```

hash桶的计算方式是目标hashcode & （数组长度 -1）

```java
final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {//这里针对String优化了Hash函数，是否使用新的Hash函数和Hash因子有关  
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
```

```txt

hashcode:0101 0101   
length-1:0000 1111
                    & 与运算 都为1才是1 获取key存储下标的时候用到
------------------
		      0101	 得到得结果肯定是小于等于length-1


hashcode是一个int类型数字 int类型占用4个字节 一个字节是8byte  hashcode也就是有32位长度
h & (length-1); 只有到了低位，高位没有用到导致散列不均匀 因为采用右移运算(高位补0)，
                    ^ 异或运算 相同为0 不同为1

充分运用高位进行异或最终再与length-1进行与运算来解决散列的问题
hashcode:0101 0101   
length-1:0000 1111
                    & 与运算 都为1才是1
------------------

                    | 或运算 都为0才是0 确定数组容量的时候用到
```

### 1.1.5 头插法

采用头插法，据说是考虑到热点数据的原因，即最近插入的元素也很可能最近会被使用到。所以为了缩短链表查找元素的时间，所以每次都会将新插入的元素放到表头

```java
void addEntry(int hash, K key, V value, int bucketIndex) {   //添加一个元素到链表中
    if ((size >= threshold) && (null != table[bucketIndex])) {  //大于等于阈值并且数组里面要有值的情况下，就进行扩容，以2的倍数
        resize(2 * table.length);  //扩容
        hash = (null != key) ? hash(key) : 0;  //因为之前key为null，添加元素也是调用了addEntry方法，所以这个对key进行了判null。有没有人会问，这个为什么又进行hash计算，因为数组扩容了。
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);//头插法
    size++;
}
```

### 1.1.6 put

Integer.highestOneBit

```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);//数组容量初始化
    }
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {//遍历数组位置的链表所有元素 如果之前已经存在了，则进行覆盖，返回覆盖的值
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);//追加元素
    return null;
}
```

```java
if (table == EMPTY_TABLE) {
    inflateTable(threshold);
}

private void inflateTable(int toSize) {
    // Find a power of 2 >= toSize
    int capacity = roundUpToPowerOf2(toSize);
}

//找大于等于阈值的2的幂
private static int roundUpToPowerOf2(int number) {
    // assert number >= 0 : "number must be non-negative";
    return number >= MAXIMUM_CAPACITY
        ? MAXIMUM_CAPACITY
        : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
}

//进制表示整合只有一个1的时候 将高位后面的全部移走 直至剩余一个高位1
public static int highestOneBit(int i) {
    // HD, Figure 3-1
    i |= (i >>  1);
    i |= (i >>  2);
    i |= (i >>  4);
    i |= (i >>  8);
    i |= (i >> 16);
    return i - (i >>> 1);
}

```

### 1.1.7 resize扩容

大于等于阈值并且数组里面要有值的情况下，就进行扩容，以2的倍数

```java
void addEntry(int hash, K key, V value, int bucketIndex) {   //添加一个元素到链表中
    if ((size >= threshold) && (null != table[bucketIndex])) {  //大于等于阈值并且数组里面要有值的情况下，就进行扩容，以2的倍数
        resize(2 * table.length);  //扩容
        hash = (null != key) ? hash(key) : 0;  //因为之前key为null，添加元素也是调用了addEntry方法，所以这个对key进行了判null。有没有人会问，这个为什么又进行hash计算，因为数组扩容了。
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

void resize(int newCapacity) {  //扩容算法
    Entry[] oldTable = table;   
    int oldCapacity = oldTable.length;  //记录老表的长度
    if (oldCapacity == MAXIMUM_CAPACITY) {  //不能超过最大限制值
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));  //将老表转成新表
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

void transfer(Entry[] newTable, boolean rehash) {   //转换函数
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);//计算新容器的数组下标
            e.next = newTable[i];  //使用头插法插入数据，也就是说如果老表中存在hash值一样三个值a,b,c ，链的顺序是a->b->c，到新的表里面hash值也一样的话,数值的顺序变成了c->b->a
            newTable[i] = e;
            e = next;
        }
    }
}

```

### 1.1.8 get

```java
public V get(Object key) {  //获取值
    if (key == null)
        return getForNullKey();  //如果key为空，
    Entry<K,V> entry = getEntry(key);  //获取值

    return null == entry ? null : entry.getValue();
}

final Entry<K,V> getEntry(Object key) {  
    if (size == 0) {  //表的长度为0 直接返回size，里面有些东西没有讲，size用的是volatile修饰，如果在获取的时候有一个线程把数据删除了
        return null;
    }

    int hash = (key == null) ? 0 : hash(key);
    for (Entry<K,V> e = table[indexFor(hash, table.length)];  //通过数组下标找到值
         e != null;  
         e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))  //一步步查找，找到key相等的就将值返回
            return e;
    }
    return null;
}

```


## 1.2 HashMap源码1.8

7和8的hashmap到底有哪些不同：
1. hash的取值算法不同
2. 求数组下标的算法不同
3. 1.8的实体是Node继承了entry，链表长度大于8的时候转换为红黑树。扩容后的计算方式 1.7 重新计算 1.8 原位置 or 原位置 + 旧容量

## 1.3 红黑树原理

## 1.4 ConcurrentHashMap源码1.7

![](../../assets/_images/java/map/jdk7chashmap.png)

### 1.4.1 成员变量

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V> implements ConcurrentMap<K, V>, Serializable {
 
	/**
	 * 在构造函数未指定初始大小时，默认使用的map大小
	 */
	static final int DEFAULT_INITIAL_CAPACITY = 16;
 
	/**
	 * 默认的扩容因子，当初始化构造器中未指定时使用。
	 */
	static final float DEFAULT_LOAD_FACTOR = 0.75f;
 
	/**
	 * 默认的并发度，这里所谓的并发度就是能同时操作ConcurrentHashMap（后文简称为chmap）的线程的最大数量，
	 * 由于chmap采用的存储是分段存储，即多个segement，加锁的单位为segment，所以一个cmap的并行度就是segments数组的长度，
	 * 故在构造函数里指定并发度时同时会影响到cmap的segments数组的长度，因为数组长度必须是大于或等于并行度的最小的2的幂。
	 */
	static final int DEFAULT_CONCURRENCY_LEVEL = 16;
 
	/**
	 * 最大容量
	 */
	static final int MAXIMUM_CAPACITY = 1 << 30;
 
	/**
	 * 每个segment的HashEntry最小容量
	 */
	static final int MIN_SEGMENT_TABLE_CAPACITY = 2;
 
	/**
	 * 分段最大的容量(最大的segment的数量)
	 */
	static final int MAX_SEGMENTS = 1 << 16; // slightly conservative
 
	/**
	 * 默认自旋次数，超过这个次数直接加锁，防止在size方法中由于不停有线程在更新map
	 * 导致无限的进行自旋影响性能，当然这种会导致ConcurrentHashMap使用了这一规则的方法 如size、clear是弱一致性的。
	 */
	static final int RETRIES_BEFORE_LOCK = 2;
 
	/**
	 * 用于索引segment的掩码值，key哈希码的高位用于选择segment
	 */
	final int segmentMask;
 
	/**
	 * 用于索引segment偏移值
	 */
	final int segmentShift;
 
	/**
	 * Segment数组
	 */
	final Segment<K, V>[] segments;
 
	transient Set<K> keySet;
	transient Set<Map.Entry<K, V>> entrySet;
	transient Collection<V> values;
}
```

Segment类和HashEntry类

```java
static final class Segment<K, V> extends ReentrantLock implements Serializable {
 
	/**
	 * scanAndLockForPut中自旋循环获取锁的最大自旋次数。
	 */
	static final int MAX_SCAN_RETRIES = Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
 
	/**
	 * 链表数组，数组中的每一个元素代表了一个链表的头部
	 */
	transient volatile HashEntry<K, V>[] table;
 
	/**
	 * 用于记录每个Segment桶中键值对的个数
	 */
	transient int count;
 
	/**
	 * 对table的大小造成影响的操作的数量（比如put或者remove操作）
	 */
	transient int modCount;
 
	/**
	 * 阈值，Segment里面元素的数量超过这个值依旧就会对Segment进行扩容
	 */
	transient int threshold;
 
	/**
	 * 负载因子，用于确定threshold，默认是1
	 */
	final float loadFactor;
}
 
static final class HashEntry<K, V> {
	final int hash;
	final K key;
	volatile V value; //为了确保读操作能够看到最新的值，将value设置成volatile
	volatile HashEntry<K, V> next; //不再用final关键字，采用unsafe操作保证并发安全
}
```

### 1.4.2 构造方法

```java
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    
    //ssize 为segments数组长度，根据concurrentLevel计算得出
    int ssize = 1;
    //依据给定的concurrencyLevel并行度，找到最适合的segments数组的长度，
    // 为上文默认并行度参数说明的大于等于concurrencyLevel的最小的2的n次方
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    
    //segmentShift和segmentMask这两个变量在定位segment时会用到，后面会详细讲
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    
    //计算cap的大小，即Segment中HashEntry的数组长度，cap也一定为2的n次方
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1; // table[]数组大小
    
    //创建segments数组并初始化第一个Segment，其余的Segment延迟初始化
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

### 1.4.3 put方法

```java
// ConcurrentHashMap类的put()方法
public V put(K key, V value) {
    Segment<K,V> s;
    //concurrentHashMap不允许key/value为空
    if (value == null)
        throw new NullPointerException();
    //hash函数对key的hashCode重新散列，避免差劲的不合理的hashcode，保证散列均匀
    int hash = hash(key);
    //返回的hash值无符号右移segmentShift位与段掩码进行位运算，定位segment
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    // 调用Segment类的put方法
    return s.put(key, hash, value, false);  
}
 
// Segment类的put()方法
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 加锁
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value); //如果加锁失败，则调用该方法
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // 根据hash计算在table[]数组中的位置
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) { //若不为null，则持续查找，知道找到key和hash值相同的节点，将其value更新
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else { //如果在链表中没有找到对应的node
                if (node != null) //如果scanAndLockForPut方法中已经返回的对应的node，则将其插入first之前
                    node.setNext(first);
                else //否则，new一个新的HashEntry
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                // 判断table[]是否需要扩容，并通过rehash()函数完成扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else  //设置node到Hash表的index索引处
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

### 1.4.4 get方法

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    
    //先定位Segment，再定位HashEntry
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

### 1.4.5 size方法

```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            if (retries++ == RETRIES_BEFORE_LOCK) {
                // 给所有的segment桶加锁
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```

### 1.4.6 总结



## 1.5 ConcurrentHashMap源码1.8

## 1.6 LinkedHashMap源码

