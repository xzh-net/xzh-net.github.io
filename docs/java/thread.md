
# 线程池源码

## 1. 线程池

队列分为：阻塞式队列(有界)、非阻塞式队列(无界),遵循着先进先出。阻塞队列与非阻塞队列区别：
-  1.非阻塞式队列超出队列总数会丢失。
-  2.阻塞式队列超出总数会进入等待（等待时间=设置超时时间）。
-  3.获取队列方面：非阻塞式队列，如果为空返回null。阻塞式队列，如果为空也会进入等待。

为什么要使用线程池：
-  1、系统执行多任务时，会为每个任务创建对应的线程，当任务执行结束之后会销毁对应的线程，在这种情况下对象被频繁的创建和销毁。
-  2、当对线程象被频繁时会占用大量的系统资源，在并发的过程中会造成资源竞争出现问题。大量的创建线程还会造成混乱，没有一个统一的管理机制，容易造成应用卡顿。
-  3、大量线程对象被频繁销毁，将会频繁出发GC机制，从而降低性能。

引入线程池的好处：
   - 1、重用线程池中的线程，避免因频繁创建和销毁线程造成的性能消耗。
   - 2、更加有效的控制线程的最大并发数，防止线程过多抢占资源造成的系统阻塞。
   - 3、对线程进行有效的管理。
   - 4、减少用户态到内核态的转换


## 2. 创建线程的四种方式
- 继承Thread类
```java
class tThread extends Thread {
    public tThread(String st) {
        super(st);
    }
    public void run(){
        for (int i = 0; i < 10; ++i){
            System.out.println(i + " " + this.getName());
            try{
                this.sleep((int)Math.random()*10);
            }
            catch(Exception e){
                System.out.println(e.toString());
            }
             
        }
    }
}
public class gbx12{
    public static void main(String args[]) {
        tThread r1 = new tThread("ME");
        tThread r2 = new tThread("You");
        r1.start();
        r2.start();
    }
}
```

- 实现Runable接口
```java
class myRunnable implements Runnable{
    @Override
    public void run() {
        for(int i =0;i<100;i++){
            if(i%2==0){
                System.out.println(i);
            }
        }
    }
}
public class threadTest2 {
    public static void main(String[] args) {
        myRunnable m = new myRunnable();
        Thread t = new Thread(m);
        t.start();
    }
}
```

- 实现Callabe接口
```java
public class Test {
	public static void main(String[] args) throws Exception {
		//创建任务
		MyCallable mc=new MyCallable();
		/**FutureTask同时实现了Runnable，Future接口。
		 * 它既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值
		 */
		//交付任务管理
		FutureTask task=new FutureTask<>(mc);//可以看成FutureTask实现了runnable接口
		Thread t=new Thread(task);
		t.start();
		System.out.println("获取结果:"+task.get());
		System.out.println("任务是否完成:"+task.isDone());
	}
}
//实现Callable接口，重写call方法
class MyCallable implements Callable {
	@Override
	public Object call() throws Exception {
		String [] str= {"apple","pear","banana","orange","grape"};
		int i=(int)(Math.random()*5);
		return str[i];
	}
}
``` 

- 线程池

ThreadPoolExecutor中定义了一个volatile变量
```java
volatile int runState;
static final int RUNNING    = 0;
static final int SHUTDOWN   = 1;
static final int STOP       = 2;
static final int TERMINATED = 3;
```

在ThreadPoolExecutor类中，最核心的任务提交方法是execute()方法，虽然通过submit也可以提交任务，但是实际上submit方法里面最终调用的还是execute()方法，所以我们只需要研究execute()方法的实现原理即可：
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
        if (runState == RUNNING && workQueue.offer(command)) {
            if (runState != RUNNING || poolSize == 0)
                ensureQueuedTaskHandled(command);
        }
        else if (!addIfUnderMaximumPoolSize(command))
            reject(command); // is shutdown or saturated
    }
}
```

接着是这句，这句要好好理解一下：

>if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command))

由于是或条件运算符，所以先计算前半部分的值，如果线程池中当前线程数不小于核心池大小，那么就会直接进入下面的if语句块了。

如果线程池中当前线程数小于核心池大小，则接着执行后半部分，也就是执行

>addIfUnderCorePoolSize(command)

如果执行完addIfUnderCorePoolSize这个方法返回false，则继续执行下面的if语句块，否则整个方法就直接执行完毕了。

如果执行完addIfUnderCorePoolSize这个方法返回false，然后接着判断：

>if (runState == RUNNING && workQueue.offer(command))

如果当前线程池处于RUNNING状态，则将任务放入任务缓存队列；如果当前线程池不处于RUNNING状态或者任务放入缓存队列失败，则执行：

>addIfUnderMaximumPoolSize(command)

如果执行addIfUnderMaximumPoolSize方法失败，则执行reject()方法进行任务拒绝处理。

我们接着看2个关键方法的实现：addIfUnderCorePoolSize和addIfUnderMaximumPoolSize：

```java
private boolean addIfUnderCorePoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (poolSize < corePoolSize && runState == RUNNING)
            t = addThread(firstTask);        //创建线程去执行firstTask任务   
        } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

我们再看addIfUnderMaximumPoolSize方法的实现，这个方法的实现思想和addIfUnderCorePoolSize方法的实现思想非常相似，唯一的区别在于addIfUnderMaximumPoolSize方法是在线程池中的线程数达到了核心池大小并且往任务队列中添加任务失败的情况下执行的

```java
private boolean addIfUnderMaximumPoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (poolSize < maximumPoolSize && runState == RUNNING)
            t = addThread(firstTask);
    } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

```java
private boolean addIfUnderCorePoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (poolSize < corePoolSize && runState == RUNNING)
            t = addThread(firstTask);        //创建线程去执行firstTask任务   
        } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

总结:

- 如果当前线程池中的线程数目小于corePoolSize，则每来一个任务，就会创建一个线程去执行这个任务；
- 如果当前线程池中的线程数目>=corePoolSize，则每来一个任务，会尝试将其添加到任务缓存队列当中，若添加成功，则该任务会等待空闲线程将其取出去执行；若添加失败（一般来说是任务缓存队列已满），则会尝试创建新的线程去执行这个任务；
- 如果当前线程池中的线程数目达到maximumPoolSize，则会采取任务拒绝策略进行处理；
- 如果线程池中的线程数量大于 corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止，直至线程池中的线程数目不大于corePoolSize；如果允许为核心池中的线程设置存活时间，那么核心池中的线程空闲时间超过keepAliveTime，线程也会被终止。


## 3. newCachedThreadPool
可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
```java
    //可缓存、定时、定长、单例 
    //SynchronousQueue
    ExecutorService executorService = Executors.newCachedThreadPool();
    for (int i = 0; i <10 ; i++) {
        final int i1 = i;
        executorService.execute(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName()+",i:"+ i1);
            }
        });
    }
```

## 4. newFixedThreadPool 定长线程池
定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
```java
    //可定长线程,核心线程5个,最多创建5个线程 (只会创建5个线程,其他线程共享这5个线程)
    //LinkedBlockingQueue
    ExecutorService executorService = Executors.newFixedThreadPool(5);
    for (int i = 0; i <10 ; i++) {
        final int i1 = i;
        executorService.execute(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName()+",i:"+ i1);
            }
        });
    }
```

## 5. newScheduledThreadPool 可定时线程池
可定时线程池，支持定时及周期性任务执行。
```java
    //可定时线程 =>核心线程数3 (延迟三秒执行)
    //DelayedWorkQueue
    long l = System.currentTimeMillis();
    ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(3);
    for (int i = 0; i <10 ; i++) {
        final int i1 = i;
        scheduledExecutorService.schedule(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName()+",i:"+ i1);
                System.out.println("耗时："+ (System.currentTimeMillis() -l)/1000 +"秒" );
            }
        },3, TimeUnit.SECONDS);
    }
```

## 6. newSingleThreadExecutor 单例线程池
单例线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
```java
    //单例线程 =>核心线程数1 最大线程数1
    //LinkedBlockingQueue
    ExecutorService executorService = Executors.newSingleThreadExecutor();
    for (int i = 0; i <10 ; i++) {
        final int i1 = i;
        executorService.execute(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName()+",i:"+ i1);
            }
        });
    }
```

## 7. 自定义线程池
```java
 public ThreadPoolExecutor(int corePoolSize,//核心线程数
                              int maximumPoolSize,//最大线程数
                              long keepAliveTime,//空闲线程的存活时间
                              TimeUnit unit,//时间单位
                              BlockingQueue<Runnable> workQueue,//任务队列
                              ThreadFactory threadFactory,//线程工厂
                              RejectedExecutionHandler handler)//拒绝策略 默认AbortPolicy
```

## 8. 如何合理配置线程池的大小

如果是CPU密集型任务，就需要尽量压榨CPU，参考值可以设为 NCPU+1

如果是IO密集型任务，参考值可以设置为2*NCPU

## 9. Fork Join源码解析和使用场景

## 10. Disruptor

序列号%数组大小=所需元素角标