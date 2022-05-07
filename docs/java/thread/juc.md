# 并发常用工具类

## 1. 线程通讯几种方式
- 1、`synchronized` 通过 wait/notify 休眠唤醒方式
- 2、`ReentrantLock` 通过 await/signal Condition方式 
- 3、`CountDownLatch` 闭锁方式
- 4、`CyclicBarrier` 栅栏的方式
- 5、`Semaphore` 信号量的方式
  - join的使用
  - yield的使用

### 1.1 synchronized基偶数打印

```java
public class addEvenDemo {
	private int i = 0;// 打印的数
	private Object obj = new Object();
	public void odd() {
		// 小于10打印
		while (i < 10) {
			synchronized (obj) {
				if (i % 2 == 1) {
					System.out.println("奇数" + i);
					i++;
					obj.notify();
				} else {
					try {
						obj.wait();
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					} // 等待偶数打印完毕
				}
			}
		}
	}
	public void even() {
		// 小于10打印
		while (i < 10) {
			synchronized (obj) {
				if (i % 2 == 0) {
					System.out.println("偶数" + i);
					i++;
					obj.notify();
				} else {
					try {
						obj.wait();
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					} // 等待奇数打印完毕
				}
			}
		}
	}
	public static void main(String[] args) {
		addEvenDemo addEvenDemo = new addEvenDemo();
		new Thread(() -> {
			addEvenDemo.odd();
		}).start();
		new Thread(() -> {
			addEvenDemo.even();
		}).start();
	}
}
```

### 1.2 ReentrantLock基偶数打印

```java
private int i = 0;// 打印的数
	private Lock lock = new ReentrantLock();
	private Condition condition = lock.newCondition();
	public void odd() {
		// 小于10打印
		while (i < 10) {
			lock.lock();
			try {
				if (i % 2 == 1) {
					System.out.println("奇数" + i);
					i++;
					condition.signal();
				} else {
					condition.await();
				}
			} catch (Exception e) {
			} finally {
				lock.unlock();
			}
		}
	}
	public void even() {
		// 小于10打印
		while (i < 10) {
			lock.lock();
			try {
				if (i % 2 == 0) {
					System.out.println("偶数" + i);
					i++;
					condition.signal();
				} else {
					condition.await();
				}
			} catch (Exception e) {
			} finally {
				lock.unlock();
			}
		}
	}
```

## 2. CountDownLatch源码解析和使用场景
- countDownLatch这个类使一个线程等待其他线程各自执行完毕后再执行。
- 是通过一个计数器来实现的，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了。

CountDownLatch: 一个线程(或者多个)， 等待另外N个线程完成某个事情之后才能执行。

```java
//14个分中心都各自算完以后 再计算排名
package com.xuzhihao.test;

import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {

	private CountDownLatch countDownLatch = new CountDownLatch(2);

	// 分中心计算
	public void fzx(String fzxId) {
		System.out.println(fzxId + "正在计算...");
		try {
			if (fzxId.equals("100")) {
				Thread.sleep(2000L);
			} else {
				Thread.sleep(5000L);
			}
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println(fzxId+"计算完毕");
		countDownLatch.countDown();
	}

	// 总分计算
	public void cal() {
		// 等待所有分中心计算完毕
		String name = Thread.currentThread().getName();
		System.out.println("等待计算排名...");
		try {
			countDownLatch.await();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("各分中心计算完毕，总排名开始计算");
	}

	public static void main(String[] args) {
		CountDownLatchDemo countDownLatchDemo = new CountDownLatchDemo();
		new Thread(() -> {
			countDownLatchDemo.fzx("100");
		}).start();

		new Thread(() -> {
			countDownLatchDemo.fzx("200");
		}).start();

		new Thread(() -> {
			countDownLatchDemo.cal();
		}).start();

	}

}

```

## 3. CyclicBarrier源码解析和使用场景

CyclicBrrier: N个线程相互等待，任何一个线程完成之前，所有的线程都必须等待

```java
package com.xuzhihao.test;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {

	private CyclicBarrier cyclicBarrier = new CyclicBarrier(3);

	// 考核项组数据生成 必须同时准备好才能一起同步
	public void check(String propertyGroup) {
		try {
			if ("BM100".equals(propertyGroup)) {
				Thread.sleep(3000);
			} else if ("BM200".equals(propertyGroup)) {
				Thread.sleep(2000);
			} else {
				Thread.sleep(1000);
			}
			System.out.println(propertyGroup + "处理完成");
			cyclicBarrier.await();
		} catch (InterruptedException | BrokenBarrierException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println(propertyGroup + "开始同步...");
	}

	public static void main(String[] args) {
		CyclicBarrierDemo cyclicBarrierDemo = new CyclicBarrierDemo();
		new Thread(() -> {
			cyclicBarrierDemo.check("BM100");
		}).start();

		new Thread(() -> {
			cyclicBarrierDemo.check("BM200");
		}).start();

		new Thread(() -> {
			cyclicBarrierDemo.check("BM300");
		}).start();

	}

}

```

## 4. Semphore源码解析和使用场景

Semaphore 只是对资源并发访问的线程数进行监控，并不会保证线程安全

- 控制对某组资源的访问权限、秒杀车位、流量控制,一组线程40个，每10个一组执行
- 可用于流量控制，限制最大的并发访问数


```java
package com.xuzhihao.test;

import java.util.concurrent.Semaphore;

public class SemaphoreDemo {

	static class TaskThread extends Thread {

		Semaphore semaphore;

		public TaskThread(Semaphore semaphore) {
			this.semaphore = semaphore;
		}

		@Override
		public void run() {
			try {
				semaphore.acquire();
				System.out.println(getName() + " 开始工作");
				Thread.sleep(1000);
				semaphore.release();
				System.out.println(getName() + " 结束 ");
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String[] args) {
		int threadNum = 5;
		Semaphore semaphore = new Semaphore(2);
		for (int i = 0; i < threadNum; i++) {
			new TaskThread(semaphore).start();
		}
	}

}

```

## 5. AtomicBoolean、AtomicInteger、AtomicLong(基本类型)

```java
public class AtomicBooleanTest {
	private static AtomicBoolean initialized = new AtomicBoolean(false);

	public void init() {
		if (initialized.compareAndSet(false, true)) {
			System.out.println("这里放置初始化代码");
		}
	}

	public static void main(String[] args) {
		AtomicBooleanTest t = new AtomicBooleanTest();
		new Thread(() -> {
			t.init();
		}).start();
		new Thread(() -> {
			t.init();
		}).start();
	}
}
```
```java
public class AtomicIntegerTest {
	public static void main(String[] args) {
		 AtomicInteger atomicInteger = new AtomicInteger(123);
	        System.out.println(atomicInteger.get());
	        int expectedValue = 123;
	        int newValue      = 234;
	        Boolean b =atomicInteger.compareAndSet(expectedValue, newValue);
	        System.out.println(b);
	        System.out.println(atomicInteger);
	}
}
```
>LongAdder的基本思路就是分散热点，将value值的新增操作分散到一个数组中，不同线程会命中到数组的不同槽中，各个线程只对自己槽中的那个value值进行CAS操作，这样热点就被分散了，冲突的概率就小很多

```java
/**
 * Atomic和LongAdder耗时测试
 *
 */
public class AtomicLongAdderTest {
	public static void main(String[] args) throws Exception {
		testAtomicLongAdder(1, 10000000);
		testAtomicLongAdder(10, 10000000);
		testAtomicLongAdder(100, 10000000);
	}

	static void testAtomicLongAdder(int threadCount, int times) throws Exception {
		System.out.println("threadCount: " + threadCount + ", times: " + times);
		long start = System.currentTimeMillis();
		testLongAdder(threadCount, times);
		System.out.println("LongAdder 耗时：" + (System.currentTimeMillis() - start) + "ms");
		System.out.println("threadCount: " + threadCount + ", times: " + times);
		long atomicStart = System.currentTimeMillis();
		testAtomicLong(threadCount, times);
		System.out.println("AtomicLong 耗时：" + (System.currentTimeMillis() - atomicStart) + "ms");
		System.out.println("----------------------------------------");
	}

	static void testAtomicLong(int threadCount, int times) throws Exception {
		AtomicLong atomicLong = new AtomicLong();
		List<Thread> list = new ArrayList<Thread>();
		for (int i = 0; i < threadCount; i++) {
			list.add(new Thread(() -> {
				for (int j = 0; j < times; j++) {
					atomicLong.incrementAndGet();
				}
			}));
		}
		for (Thread thread : list) {
			thread.start();
		}
		for (Thread thread : list) {
			thread.join();
		}
		System.out.println("AtomicLong value is : " + atomicLong.get());
	}

	static void testLongAdder(int threadCount, int times) throws Exception {
		LongAdder longAdder = new LongAdder();
		List<Thread> list = new ArrayList<Thread>();
		for (int i = 0; i < threadCount; i++) {
			list.add(new Thread(() -> {
				for (int j = 0; j < times; j++) {
					longAdder.increment();
				}
			}));
		}
		for (Thread thread : list) {
			thread.start();
		}
		for (Thread thread : list) {
			thread.join();
		}
		System.out.println("LongAdder value is : " + longAdder.longValue());
	}
}
```

## 6. AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray（数组类型）



## 7. AtomicReference、AtomicStampedRerence、AtomicMarkableReference（引用类型）



## 8. AtomicIntegerFieldUpdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater（属性原子修改器）
