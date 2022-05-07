
# Queue源码

BlockingQueue的核心方法：
```java
public interface BlockingQueue<E> extends Queue<E> {
    //将给定元素设置到队列中，如果设置成功返回true, 否则抛出异常。如果是往限定了长度的队列中设置值，推荐使用offer()方法。
    boolean add(E e);

    //将给定的元素设置到队列中，如果设置成功返回true, 否则返回false. e的值不能为空，否则抛出空指针异常。
    boolean offer(E e);

    //将元素设置到队列中，如果队列中没有多余的空间，该方法会一直阻塞，直到队列中有多余的空间。
    void put(E e) throws InterruptedException;

    //将给定元素在给定的时间内设置到队列中，如果设置成功返回true, 否则返回false.
    boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException;

    //从队列中获取值，如果队列中没有值，线程会一直阻塞，直到队列中有值，并且该方法取得了该值。
    E take() throws InterruptedException;

    //在给定的时间里，从队列中获取值，时间到了直接调用普通的poll方法，为null则直接返回null。
    E poll(long timeout, TimeUnit unit) throws InterruptedException;

    //获取队列中剩余的空间。
    int remainingCapacity();

    //从队列中移除指定的值。
    boolean remove(Object o);

    //判断队列中是否拥有该值。
    public boolean contains(Object o);

    //将队列中值，全部移除，并发设置到给定的集合中。
    int drainTo(Collection<? super E> c);

    //指定最多数量限制将队列中值，全部移除，并发设置到给定的集合中。
    int drainTo(Collection<? super E> c, int maxElements);
}
```

操作 | 抛出异常 | 特殊值 | 阻塞请求 | 超时
----|----|----|----|----
插入、添加 | add(e) | offer(e) | put(e) | offer(e, time, unit)
移除、获取 | remove() | poll() | take() | poll(time, unit)
检查 | element() | peek() | 不可用 | 不可用




## 1.1 ArrayBlockingQueue 数组阻塞队列
- 数据结构：静态数组固定长度无扩容机制
- 存取同一把锁互斥 操作同一个对象
- 阻塞：出队为0时，无元素可取，阻塞在notEmpty 入队为数组长度时，无法刚入，阻塞在notFull
- 入队：队首添加元素记录putIndex 唤醒 notEmpty 
- 出队：队首获取元素记录takeIndex 唤醒 notFull
- 先进先出 读写互相排斥

## 1.2 LinkedBlockingQueue 链表阻塞队列
- 数据结构：链表 可以指定容量默认Integer.MAX_VALUE
- 锁分离存取互不排斥takeLock、putLock
- 阻塞同ArrayBlockingQueue
- 入队：队尾入队 记录last节点
- 出队：队首出队 记录head节点
- 删除的时候两把锁一起加
- 先进先出
  
## 1.3 SynchronousQueue 同步队列、无缓冲阻塞队列
- 存取和调用都使用transfer方法
  - put、offer为生产者，携带了数据e,为Data模式1，设置到QNode中
  - tale、pool为消费者，不携带数据e,为REQUEST模式0，设置到QNode中
- 存取和调用都使用transfer方法
- 数据结构：链表Node
- 锁：CAS + 自旋
- 阻塞：`LockSupport`
- 公平模式：TransferQueue 队尾入列 对头出列 先进先出
- 非公平模式：TransferStack 栈顶入列 栈顶出列 后进先出
```java
//默认false非公平
public SynchronousQueue() {
    this(false);
}
//公平模式使用TransferQueue先进先出
//非公平模式使用TransferStack后进先出
public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }
```

## 1.4 LinkedTransferQueue 由链表结构组成的无界阻塞
- 数据结构：链表Node
- 锁：CAS + 自旋
- 阻塞：`LockSupport`
- 可以看作LinkedBlockingQueue、SynchronousQueue（公平模式）、ConcurrentLinkedQueue三者的集合体
- 对于入队之后，先自旋一定次数后再调用LockSupport.park()或LockSupport.parkNanos阻塞

## 1.5 PriorityBlockingQueue 优先级阻塞队列 
- 数据结构：数组+平衡二叉堆 小堆顶 
- 可以指定容量，自动扩容，容量小于64则翻倍，大于64增加一半，最大容量Integer.MAX_VALUE
- 锁：ReentrantLock存取同一把锁
- 阻塞：notEmpty 出队队列为空的时候阻塞
- 入队：不阻塞，永远返回成功，无界。有比较器的时候根据比较器进行堆化，没有使用默认比较器，自下而上堆化
- 出队：弹出堆顶元素，自上而下重新堆化，为空则阻塞

## 1.6 DelayQueue 延迟队列
- 数据结构：Priority 与PriorityBlockingQueue类似，没有阻塞功能
- 锁：ReentrantLock
- 阻塞：Condition available
- 入队：不阻塞，与优先级队列相同，唤醒available
- 出队：为空的时候阻塞available。检测堆顶元素过期时间，小于等于0则取出，大于0阻塞，判断leader线程是否为空
  
## 1.7 LinkedBlockingDeque 链表阻塞双端队列
- 数据结构：同LinkedBlockingQueue
- 锁定：同ArrayBlockingQueue
- 阻塞：同ArrayBlockingQueue
- 入队：队首队尾都可
- 出队：队首队尾都可
