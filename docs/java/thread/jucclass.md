# java.util.concurrent

类名 | 描述
----|----
`atomic包下(17)` | 
AtomicBoolean | 可以用原子方式更新的 boolean 值
AtomicInteger | 可以用原子方式更新的 int 值
AtomicIntegerArray | 可以用原子方式更新其元素的 int 数组
AtomicIntegerFieldUpdater<T> | 基于反射的实用工具，可以对指定类的指定 volatile int 字段进行原子更新
AtomicLong | 可以用原子方式更新的 long 值
AtomicLongArray | 可以用原子方式更新其元素的 long 数组
AtomicLongFieldUpdater<T> | 基于反射的实用工具，可以对指定类的指定 volatile long 字段进行原子更新
AtomicMarkableReference<V> | AtomicMarkableReference 维护带有标记位的对象引用，可以原子方式对其进行更新
AtomicReference<V> | 可以用原子方式更新的对象引用
AtomicReferenceArray<E> | 可以用原子方式更新其元素的对象引用数组
AtomicReferenceFieldUpdater<T,V> | 基于反射的实用工具，可以对指定类的指定 volatile 字段进行原子更新
AtomicStampedReference<V> | AtomicStampedReference 维护带有整数“标志”的对象引用，可以用原子方式对其进行更新
DoubleAccumulator | 一个或多个变量一起维护使用提供的功能更新的运行的值 double
DoubleAdder | 一个或多个变量一起保持初始为零 double和
LongAccumulator | 一个或多个变量，它们一起保持运行 long使用所提供的功能更新值
LongAdder | 一个或多个变量一起保持初始为零 long总和
Striped64 | 
`locks包下(10)` | 
AbstractOwnableSynchronizer | 可以由线程独占的同步器
AbstractQueuedLongSynchronizer | AbstractQueuedSynchronizer的一个版本，其中同步状态保持为long
AbstractQueuedSynchronizer | 提供一个框架，用于实现依赖先进先出（FIFO）等待队列的阻塞锁和相关同步器（信号量，事件等）
Condition | Lock替换synchronized方法和语句的使用， Condition取代了对象监视器方法的使用
Lock | 
LockSupport | 用于创建锁和其他同步类的基本线程阻塞原语
ReadWriteLock | 维护一对关联的locks ，一个用于只读操作，一个用于写入
ReentrantLock | 一个可重入互斥Lock具有与使用synchronized方法和语句访问的隐式监视锁相同的基本行为和语义，但具有扩展功能
ReentrantReadWriteLock | ReadWriteLock的实现支持类似的语义到ReentrantLock 
StampedLock | 一种基于能力的锁，具有三种模式用于控制读/写访问
`concurrent包下(58)` | 
BlockingDeque | 
BlockingQueue | 
BrokenBarrierException | 
Callable | 
CancellationException | 
CompletableFuture | 
CompletionException | 
CompletionService | 
CompletionStage | 
ConcurrentHashMap | 
ConcurrentLinkedDeque | 
ConcurrentLinkedQueue | 
ConcurrentMap | 
ConcurrentNavigableMap | 
ConcurrentSkipListMap | 
ConcurrentSkipListSet | 
CopyOnWriteArrayList | 
CopyOnWriteArraySet | 
CountDownLatch | 
CountedCompleter | 
CyclicBarrier | 
Delayed | 
DelayQueue | 
Exchanger | 
ExecutionException | 
Executor | 
ExecutorCompletionService | 
Executors | 
ExecutorService | 
ForkJoinPool | 
ForkJoinTask | 
ForkJoinWorkerThread | 
Future | 
FutureTask | 
LinkedBlockingDeque | 
LinkedBlockingQueue | 
LinkedTransferQueue | 
Phaser | 
PriorityBlockingQueue | 
RecursiveAction | 
RecursiveTask | 
RejectedExecutionException | 
RejectedExecutionHandler | 
RunnableFuture | 
RunnableScheduledFuture | 
ScheduledExecutorService | Netty的EventLoop扩展于此
ScheduledFuture | 
ScheduledThreadPoolExecutor | 
Semaphore | 
SynchronousQueue | 
ThreadFactory | 
ThreadLocalRandom | 
ThreadPoolExecutor | 
TimeoutException | 
TimeUnit | 
TransferQueue | 
