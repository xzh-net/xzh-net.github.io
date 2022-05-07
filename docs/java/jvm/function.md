#  基础算法函数

pom依赖
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<artifactId>test</artifactId>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.3.0.RELEASE</version>
		<relativePath />
	</parent>
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<java.version>1.8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>cn.hutool</groupId>
			<artifactId>hutool-all</artifactId>
			<version>4.5.7</version>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
		<dependency>
			<groupId>org.openjdk.jol</groupId>
			<artifactId>jol-core</artifactId>
			<version>0.9</version>
		</dependency>
	</dependencies>
	<build>
		<pluginManagement>
			<plugins>
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-compiler-plugin</artifactId>
					<version>2.3.2</version>
					<configuration>
						<source>1.8</source>
						<target>1.8</target>
						<encoding>UTF-8</encoding>
					</configuration>
				</plugin>
			</plugins>
		</pluginManagement>
	</build>
</project>
```
## 1. java基础

### 1.1 Unsafe的使用

```java

package com.xuzhihao.test;
import java.lang.reflect.Field;
import sun.misc.Unsafe;
public class UnsafeTest {

	private int i = 0;
	private static sun.misc.Unsafe unsafe;
	private static long offset;
	private String[] table = { "1", "2", "3", "4" };
	static {
		try {
			Field field = Unsafe.class.getDeclaredField("theUnsafe");
			field.setAccessible(true);
			unsafe = (Unsafe) field.get(null);
			offset = unsafe.objectFieldOffset(UnsafeTest.class.getDeclaredField("i"));
		} catch (Exception e) { // Internal reference
			e.printStackTrace();
		}
	}
    //unsafe方式修改数组中值
	public static void main2(String[] args) {
		final UnsafeTest po = new UnsafeTest();
		// 数组中存储的对象的对象头大小
		int ns = unsafe.arrayIndexScale(String[].class);
		// 数组中第一个元素的起始位置
		int base = unsafe.arrayBaseOffset(String[].class);
		System.out.println(unsafe.getObject(po.table, base + 3 * ns));
	}
    //多线程并发修改
	public static void main(String[] args) {
		final UnsafeTest po = new UnsafeTest();
		new Thread(new Runnable() {
			@Override
			public void run() {
				while (true) {
					boolean b = unsafe.compareAndSwapInt(po, offset, po.i, po.i + 1);
					if (b)
						System.out.println(unsafe.getIntVolatile(po, offset));
					try {
						Thread.sleep(500);
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
				}
			}
		}).start();
		new Thread(new Runnable() {
			@Override
			public void run() {
				while (true) {
					boolean b = unsafe.compareAndSwapInt(po, offset, po.i, po.i + 1);
					if (b)
						System.out.println(unsafe.getIntVolatile(po, offset));
					try {
						Thread.sleep(500);
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
				}
			}
		}).start();
	}
}
```

### 1.2 变量交换

```java
	int a = 3, b = 5;
	a = a + b;// a==8,b==5
	b = a - b;// a==8,b==3
	a = a - b;// a==5,b==3
	System.out.println(a + "," + b);
	a = a ^ b;
	b = a ^ b;
	a = a ^ b;
	System.out.println(a + "," + b);
```

### 1.3 操作数栈与自增自减

```java
package com.xuzhihao.test;
/**
 * 关于操作数栈与自增自减的面试题
 * @author Administrator
 *
 */
//考点：
// 赋值= 最后计算
// 从左到右依次压入操作数栈
// 实际计算按照运算符优先级
// 自增 自减直接操作局部变量表的值 不经过操作数栈
// 最后赋值之前的结果 存储在操作数栈中
// JVM虚拟机规范相关指令
public class Exam1 {

	public static void main(String[] args) {
		int i = 1;
		i = i++;
		int j = i++;
		int k = i + ++i * i++;
		/**
		 * 在i = i ++ 的时候 会想把i压入操作数栈 ，压栈的解析 从来都是从左往右的 所以 1 会被压入操作数栈 ， 然后继续往右解析 i 自增1 此时 i
		 * = 2 ,但是自增自减都是直接修改变量的值 而不经过操作数栈的，所以操作数栈里的值还是等于 1， 最后进行赋值操作 i = 1,所以i
		 * 又被操作数栈里的1覆盖了 所以 i还是等于1 。
		 */
		System.out.println(i);
		/**
		 * int j = i++ 其实与 i = i++ 情况一致 ，第一步 i = 1这个值进入操作数栈 ，随后在向右走 i自增 此时 i = 2 ,随后
		 * 操作数栈里的 1 再赋值给了 j 所以 j = 1。 于是到这一步： i = 2 ; j = 1;
		 */
		System.out.println(j);
		/**
		 * 还是从左往右解析操作数栈
		 * i = 2 , 2这个值先进入操作数栈 ，随后i自增 i变成了3 随后 3又进入了操作数栈 ，最后又是i++ 因为先遇到i所以此时 i =3
		 * 又进入操作数栈 后面 i 再自增 i 变成了4，
		 * 
		 * 但并不影响操作数栈的值 所以 最终的运算为 k = 2 + 3 * 3 =11
		 */
		System.out.println(k);

	}
}
```

### 1.4 方法参数的传递机制
```java
package com.xuzhihao.test;

import java.util.Arrays;

//  考点：方法的参数传递机制： 
//  1.形参是基本数据类型 
//		传递数据值 
//  2.实参是引用数据类型 
//		传递地址值
//  	特殊类型：String,包装类等对象的不可变性
  
 
public class Exam4 {

	public static void main(String[] args) {
		int i = 1;
		String str = "hello";
		Integer num = 200;
		int[] arr = { 1, 2, 3, 4, 5 };
		MyData my = new MyData();

		change(i, str, num, arr, my);

		System.out.println("i= " + i);
		System.out.println("str= " + str);
		System.out.println("num= " + num);
		System.out.println("arr= " + Arrays.toString(arr));
		System.out.println("my.a= " + my.a);
	}

	private static void change(int j, String s, Integer n, int[] a, MyData m) {
		j += 1;
		s += "world";
		n += 1;
		a[0] += 1;
		m.a += 1;
	}

}

class MyData {
	int a = 10;
}
```

### 1.5 就近原则

```java
package com.xuzhihao.test;

//就近原则
//	变量的分类 .
//		成员变量：类变量和实例变量
//		局部变量
//	非静态代码块的执行：每次创建实例都会执行
//	方法调用规则：调用一次执行一次
public class Exam6 {
//	变量存储位置
//	局部变量：栈
//	实例变量：堆
//	类变量 ：方法区
	static int s;// 成员变量 类变量 存储在：方法区
	int i;// 成员变量 实例变量 存储在：堆
	int j;// 成员变量 实例变量 存储在：堆
	{
		int i = 1;
		i++;
		j++;
		s++;
	}

	public void test(int j) {// 形参局部变量 j
		j++;
		i++;
		s++;
	}

	public static void main(String[] args) {
		Exam6 obj1 = new Exam6();//局部变量 存储在：栈
		Exam6 obj2 = new Exam6();
		obj1.test(10);
		obj1.test(20);
		obj2.test(30);
		System.out.println(obj1.i + "," + obj1.j + "," + obj1.s);
		System.out.println(obj2.i + "," + obj2.j + "," + obj2.s);
	}
}
```

### 1.6 对象内存布局
```java
package com.xuzhihao.test;

import org.openjdk.jol.info.ClassLayout;

public class JolCoreTest {
	public static void main(String[] args) {
		// TODO Auto-generated method stub
		LockObj obj = new LockObj();
		System.out.println(ClassLayout.parseInstance(obj).toPrintable());
	}
}
class LockObj {
	private int i;
	private boolean b;

}

```

### 1.8 字符串常量池
```java
package com.xuzhihao.test;

public class StringTest {
	/**
	 * a的值在编译时就被确定下来，故其值"xy"被放入String的驻留池(驻留池在堆中)并被a指向。
	 * b的值在编译时也被确定，那么b的值在String的驻留池中找是否有等于"xy"的值，有的话也被b指向。 故两个对象地址一致
	 * 
	 * @return true
	 */
	public static Boolean testString1() {
		String a = "xy";
		String b = "xy";
		return a == b;
	}

	/**
	 * b的值在是两个常量相加，编译时也被确定。
	 * 
	 * @return true
	 */
	public static Boolean testString2() {
		String a = "xyy";
		String b = "xy" + "y";
		return a == b;
	}

	/**
	 * b的值为一个变量和一个常量相加，无法编译时被确定,而是会在堆里新生成一个值为"abc"的对象
	 * 
	 * @return false
	 */
	public static Boolean testString3() {
		String a = "xyy";
		String b = "xy";
		b = b + "y";
		return a == b;
	}

	/**
	 * b的值都无法编译时被确定,而是会在堆里分别新生成一个对象叫"xyy"。
	 * 
	 * @return false
	 */
	public static Boolean testString4() {
		String a = "xyy";
		String b = "xy".concat("y");
		return a == b;
	}

	/**
	 * new String()创建的字符串不是常量，不能在编译期就确定，所以new String() 创建的字符串不放入常量池中，它们有自己的地址空间。
	 * a,b的值都无法编译时被确定,会在堆里分别新生成一个值为"xy"的对象。
	 * 
	 * @return fasle
	 */
	public static Boolean testString5() {
		String a = new String("xy");
		String b = new String("xy");
		return a == b;
	}

	/**
	 * intern()把驻留池中"xy"的引用赋给b。
	 * 
	 * @return true
	 */
	public static Boolean testString6() {
		String a = "xy";
		String b = new String("xy");
		b = b.intern();
		return a == b.intern();
	}

	/**
	 * char的toString方法返回的是一个char对象的字符串，而不是我们想象的"xy"
	 * 
	 * @return false
	 */
	public static Boolean testString7() {
		String b = "xy";
		char[] a = new char[] { 'x', 'y' };
		return a.toString().equals(b);
	}

	/**
	 * char是一种新的类型，不存在驻留池的概念。
	 * 
	 * @return fasle
	 */
	public static Boolean testString8() {
		String b = "xy";
		char[] a = new char[] { 'x', 'y' };
		return a.toString() == b;
	}

	/**
	 * String不可变性的体现
	 */
	String str = "xy";

	public String chage(String str) {
		str = "xyy";
		return str;
	}

	public static void main(String[] args) {
		print(testString1()); // true
		print(testString2()); // true
		print(testString3()); // fasle
		print(testString4()); // false
		print(testString5()); // false
		print(testString6()); // true
		print(testString7()); // false
		print(testString8()); // false
	}

	public static void print(Object o) {
		System.out.println(o);
	}
}
```

### 1.9 ThreadLocal
```java
package com.xuzhihao.test;

public class ThreadLocalDemo {
	static class Bank {
		private ThreadLocal<Integer> threadlocal = new ThreadLocal<Integer>() {
			protected Integer initialValue() {
				return 0;
			}
		};

		public Integer get() {
			return threadlocal.get();
		}

		public void set(Integer money) {
			threadlocal.set(threadlocal.get() + money);
		}
	}

	static class Transfer implements Runnable {
		private Bank bank;

		public Transfer(Bank bank) {
			this.bank = bank;
		}

		@Override
		public void run() {
			for (int i = 0; i < 10; i++) {
				bank.set(10);
				System.out.println(Thread.currentThread().getName() + "账户余额：" + bank.get());
			}

		}
	}

	public static void main(String[] args) {
		Bank bank = new Bank();
		Transfer transfer = new Transfer(bank);
		new Thread(transfer, "客户1").start();
		new Thread(transfer, "客户2").start();
	}

}

```

## 2. 算法

### 2.1 排序算法

#### 2.1.1 冒泡排序

取第一个元素与后一个比较，如果大于后者，就与后者互换位置，不大于，就保持位置不变。再拿第二个元素与后者比较，如果大于后者，就与后者互换位置。一轮比较之后，最大的元素就移动到末尾。相当于最大的就冒出来了。再进行第二轮，第三轮，直到排序完毕

冒泡排序：比较 （N-1)+(N-2)+...+2+1 = N*(N-1)/2=N2/2

　　　　　交换  0——N2/2 = N2/4

　　　　　总时间 3/4*N2

```java
/**
* 冒泡排序
*/
public static void bubble() {
	int[] arr = { 10, 6, 3, 2, 1, 7 };
	for (int i = 0; i < arr.length - 1; i++) {// 外层循环n-1
		for (int j = 0; j < arr.length - i - 1; j++) {// 内层循环n-i-1
			if (arr[j] > arr[j + 1]) {// 从第一个开始，往后两两比较大小，如果前面的比后面的大，交换位置
				int tmp = arr[j];
				arr[j] = arr[j + 1];
				arr[j + 1] = tmp;
			}
		}
	}
	System.out.println(Arrays.toString(arr));
}
```

#### 2.1.2 插入排序

把元素分为已排序的和未排序的。每次从未排序的元素取出第一个，与已排序的元素从尾到头逐一比较，找到插入点，将之后的元素都往后移一位，腾出位置给该元素

```java
/*
* 插入排序方法
*/
public static  void doInsertSort() {
	int[] array = { 8, 5, 10, 2, 19, 1 };
	for (int index = 1; index < array.length; index++) {// 外层向右的index，即作为比较对象的数据的index
		int temp = array[index];// 用作比较的数据
		int leftindex = index - 1;
		while (leftindex >= 0 && array[leftindex] > temp) {// 当比到最左边或者遇到比temp小的数据时，结束循环
			array[leftindex + 1] = array[leftindex];
			leftindex--;
		}
		array[leftindex + 1] = temp;// 把temp放到空位上
		System.out.println(Arrays.toString(array));
	}
}
```

#### 2.1.3 选择排序

从数组中找到最小的元素，和第一个位置的元素互换。 

从第二个位置开始，找到最小的元素，和第二个位置的元素互换。

........

直到选出array.length-1个较小元素，剩下的最大的元素自动排在最后一位

```java
public static void selectrSort() {
	int[] arr = { 3, 6, 10, 2, 19, 1 };
	for (int i = 0; i < arr.length - 1; i++) {
		int minPos = i;
		for (int j = i; j < arr.length; j++) {
			if (arr[j] < arr[minPos]) {
				minPos = j;// 找出当前最小元素的位置
			}
		}
		if (arr[minPos] != arr[i]) {
			swap(arr, minPos, i);
		}
	}
}

public static void swap(int[] arr, int a, int b) {
	int temp = arr[a];
	arr[a] = arr[b];
	arr[b] = temp;
}
```

### 2.2 递归迭代
```java
package com.xuzhihao.test;

/**
 * 实现f(n):求n步台阶，一共有集中走法，每次可以走一步或者两步
 * 
 * @author admin
 *
 */

//方法调用自身称为递归，利用变量的原始值推断出新的值称为迭代

//迭代：
//	优点：执行效率高，因为时间只因为循环次数而增加，没有额外空间的开销
//	缺点：代码相对可读性不好
public class Exam5 {
	public static int f(int n) {
		if (n == 1 || n == 2) {
			return n;
		}
		return f(n - 2) + f(n - 1);
	}

	public static int loop(int n) {
		if (n == 1 || n == 2) {
			return n;
		}
		int one = 2;// 初始走两步的走法
		int two = 1;// 初始走一步的走法
		int sum = 0;
		for (int i = 3; i <= n; i++) {
			// 最后走两步+最后走一步
			sum = one + two;
			two = one;
			one = sum;
		}
		return sum;
	}

	public static void main(String[] args) {
		// 递归调用
		System.out.println(f(10));
		// 迭代调用
		System.out.println(loop(10));
	}
}

```

### 2.3 二叉树
```java
package com.xuzhihao.test;

//java BinaryTree
//二叉树
//判断一个数是不是2的幂次方
public class BinaryTree {

	int data; // 根节点数据
	BinaryTree left;
	BinaryTree right;

	/**
	 * 构造
	 * 
	 * @param data
	 */
	public BinaryTree(int data) {
		this.data = data;
		this.left = null;
		this.right = null;
	}

	public void insert(BinaryTree root, int data) {
		if (data > root.data) {// 右子树
			if (root.right == null) {
				root.right = new BinaryTree(data);
			} else {
				this.insert(root.right, data);
			}
		} else {// 左子树
			if (root.left == null) {
				root.left = new BinaryTree(data);
			} else {
				this.insert(root.left, data);
			}
		}
	}

	public static void preOrder(BinaryTree root) {//
		if (root != null) {
			System.out.print(root.data + "-");
			preOrder(root.left);
			preOrder(root.right);
		}
	}

	public static void inOrder(BinaryTree root) {
		if (root != null) {
			inOrder(root.left);
			System.out.print(root.data + "--");
			inOrder(root.right);
		}
	}

	public static void postOrder(BinaryTree root) {
		if (root != null) {
			postOrder(root.left);
			postOrder(root.right);
			System.out.print(root.data + "--");
		}
	}

	public static void main2(String[] args) {
		int[] array = { 12, 76, 35, 22, 16, 48, 90, 46, 9, 40 };
		BinaryTree root = new BinaryTree(array[0]);// 创建二叉树
		for (int i = 1; i < array.length; i++) {
			root.insert(root, array[i]);// 向二叉树中插入数据
		}
		System.out.println("先根遍历：");
		preOrder(root);
		System.out.println();
		System.out.println("中根遍历：");
		inOrder(root);
		System.out.println();
		System.out.println("后根遍历：");
		postOrder(root);
	}

	public static void main(String[] args) {
		int n = 16;
		System.out.println((n & (n - 1)) == 0);
		System.out.println(nCF3(n));
	}

	public static boolean nCF3(int n) {
		boolean boo = true;
		String s = Integer.toBinaryString(n);
		byte[] b = s.getBytes();

		for (int i = 1; i < b.length; i++) {
			if (b[i] != 48) {
				boo = false;
				break;
			}
		}
		return boo;
	}

	public static void main4(String[] args) {
		int n = 17;
		System.out.println(n % 3);
		System.out.println(n / 3);

		System.out.println(nCF(n));
	}

	public static boolean nCF(int n) {
		boolean b = false;
		while (true) {
			int j = n % 2;
			n = n / 2;
			if (j == 1) {
				b = false;
				break;
			}
			if (n == 2) {
				b = true;
				break;
			}

		}
		return b;
	}
}
```

## 3. 函数工具

### 3.1 红包发放

```java
package com.xuzhihao.wechat;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Random;

/**
 * 红包发放
 * 
 * @author Administrator
 *
 */
public class RedPackage {

	/**
	 * 二倍均值发红包算法，金额参数以分为单位
	 * 
	 * @param totalAmount    金额(分)
	 * @param totalPeopleNum 数量
	 * @return
	 */
	public static List<Integer> divideRedPackage(Integer totalAmount, Integer totalPeopleNum) {
		List<Integer> amountList = new ArrayList<Integer>();
		Integer restAmount = totalAmount;
		Integer restPeopleNum = totalPeopleNum;
		Random random = new Random();
		for (int i = 0; i < totalPeopleNum - 1; i++) {
			// 随机范围：[1，剩余人均金额的两倍)，左闭右开
			int amount = random.nextInt(restAmount / restPeopleNum * 2 - 1) + 1;
			restAmount -= amount;
			restPeopleNum--;
			amountList.add(amount);
		}
		amountList.add(restAmount);
		return amountList;
	}

	/**
	 * 线段分割算法
	 * 
	 * @param totalAmount    红包金额 分
	 * @param totalPeopleNum 红包个数
	 */
	public static List<Double> divideRedPackage2(Integer totalAmount, Integer totalPeopleNum) {
		List<Integer> list = new ArrayList<Integer>();// 线段集合
		List<Double> amountList = new ArrayList<Double>();
		Random random = new Random();
		while (list.size() <= totalPeopleNum - 2) {
			int i = random.nextInt(totalAmount) + 1;// 最低1分钱
			if (list.indexOf(i) < 0) {// 非重复切割添加到集合
				list.add(i);
			}
		}
		Collections.sort(list);
		int flag = 0;
		int fl = 0;
		for (int i = 0; i < list.size(); i++) {
			int temp = list.get(i) - flag;
			flag = list.get(i);
			fl += temp;
			amountList.add(AmountUtil.div(temp, 100));
		}
		// 最后一个红包
		list.add(totalAmount - fl);
		amountList.add(AmountUtil.div(AmountUtil.sub(totalAmount, fl), 100));
		return amountList;
	}
}
```

工具类
```java
package com.xuzhihao.wechat;

import java.math.BigDecimal;

/**
 * 金额计算工具
 * 
 * 
 * @author poll
 * @version $Id: AmountUtil.java, v 0.1 2016年11月9日 上午10:58:19 poll Exp $
 */
public class AmountUtil {

	/**
	 * 由于Java的简单类型不能够精确的对浮点数进行运算，这个工具类提供精 确的浮点数运算，包括加减乘除和四舍五入。
	 * 
	 * 一：在 Java 中写入 new BigDecimal(0.1) 但是它实际上等于
	 * 0.1000000000000000055511151231257827021181583404541015625。 这是因为 0.1 无法准确地表示为
	 * double（或者说对于该情况，不能表示为任何有限长度的二进制小数）。 这样，传入 到构造方法的值不会正好等于 0.1（虽然表面上等于该值）。
	 * 二：另一方面，String 构造方法是完全可预知的：写入 new BigDecimal("0.1") 将创建一个 BigDecimal， 它正好
	 * 等于预期的 0.1。因此，比较而言，通常建议优先使用 String 构造方法。 三：使用public static BigDecimal
	 * valueOf(double val) 使用 Double.toString(double) 方法提供的 double 规范的字符串表示形式将
	 * double 转换为 BigDecimal。 这通常是将 double（或 float）转化为 BigDecimal 的首选方法， 因为返回的值等于从构造
	 * BigDecimal（使用 Double.toString(double) 得到的结果）得到的值。
	 */

	// 默认除法运算精度
	private static final int DEF_DIV_SCALE = 2;

	// 这个类不能实例化

	private AmountUtil() {

	}

	public static BigDecimal toBigDecimal(double v) {
		return new BigDecimal(Double.valueOf(v));
	}

	public static double toDoubleValue(BigDecimal b) {
		if (b == null) {
			return 0.0;
		} else {
			return b.doubleValue();
		}
	}

	/**
	 * 金额单位转为分
	 * 
	 * @param amt
	 * @return
	 */
	public static long unitToCent(double v) {

		BigDecimal a = new BigDecimal(Double.toString(v));
		BigDecimal b = BigDecimal.valueOf(100);

		return a.multiply(b).longValue();
	}

	/**
	 * 金额单位转为元
	 * 
	 * @param amt
	 * @return
	 */
	public static double unitToYuan(long v) {

		BigDecimal b1 = new BigDecimal(Long.toString(v));

		BigDecimal b2 = new BigDecimal(Long.toString(100));

		return b1.divide(b2).doubleValue();
	}

	/** */
	/**
	 * 
	 * 提供精确的加法运算。
	 * 
	 * @param v1 被加数
	 * 
	 * @param v2 加数
	 * 
	 * @return 两个参数的和
	 * 
	 */
	public static double add(double v1, double v2) {

		BigDecimal b1 = new BigDecimal(Double.toString(v1));

		BigDecimal b2 = new BigDecimal(Double.toString(v2));

		return b1.add(b2).doubleValue();

	}

	/**
	 * 
	 * 提供精确的减法运算。
	 * 
	 * @param v1 被减数
	 * 
	 * @param v2 减数
	 * 
	 * @return 两个参数的差
	 * 
	 */
	public static double sub(double v1, double v2) {

		BigDecimal b1 = new BigDecimal(Double.toString(v1));

		BigDecimal b2 = new BigDecimal(Double.toString(v2));

		return b1.subtract(b2).doubleValue();

	}

	/**
	 * 
	 * 提供精确的乘法运算。
	 * 
	 * @param v1 被乘数
	 * 
	 * @param v2 乘数
	 * 
	 * @return 两个参数的积
	 * 
	 */
	public static double mul(double v1, double v2) {

		BigDecimal b1 = new BigDecimal(Double.toString(v1));

		BigDecimal b2 = new BigDecimal(Double.toString(v2));

		return b1.multiply(b2).doubleValue();

	}

	/** */
	/**
	 * 
	 * 提供（相对）精确的除法运算，当发生除不尽的情况时，精确到
	 * 
	 * 小数点以后2位，以后的数字四舍五入。
	 * 
	 * @param v1 被除数
	 * 
	 * @param v2 除数
	 * 
	 * @return 两个参数的商
	 * 
	 */
	public static double div(double v1, double v2) {

		return div(v1, v2, DEF_DIV_SCALE);

	}

	/**
	 * 
	 * 提供（相对）精确的除法运算。当发生除不尽的情况时，由scale参数指
	 * 
	 * 定精度，以后的数字四舍五入。
	 * 
	 * @param v1    被除数
	 * 
	 * @param v2    除数
	 * 
	 * @param scale 表示表示需要精确到小数点以后几位。
	 * 
	 * @return 两个参数的商
	 * 
	 */
	public static double div(double v1, double v2, int scale) {

		if (scale < 0) {

			throw new IllegalArgumentException("The scale must be a positive integer or zero");

		}

		BigDecimal b1 = new BigDecimal(Double.toString(v1));

		BigDecimal b2 = new BigDecimal(Double.toString(v2));

		return b1.divide(b2, scale, BigDecimal.ROUND_HALF_UP).doubleValue();

	}

	/** */
	/**
	 * 
	 * 提供精确的小数位四舍五入处理。
	 * 
	 * @param v     需要四舍五入的数字
	 * 
	 * @param scale 小数点后保留几位
	 * 
	 * @return 四舍五入后的结果
	 * 
	 */
	public static double round(double v, int scale) {

		if (scale < 0) {
			throw new IllegalArgumentException("The scale must be a positive integer or zero");

		}

		BigDecimal b = new BigDecimal(Double.toString(v));

		BigDecimal one = new BigDecimal("1");

		return b.divide(one, scale, BigDecimal.ROUND_HALF_UP).doubleValue();

	}

	/**
	 * if a==b return true, else return false;
	 * 
	 * @param a
	 * @param b
	 * @return
	 */
	public static boolean equal(double a, double b) {
		if ((a - b > -0.001) && (a - b) < 0.001) {
			return true;
		} else {
			return false;
		}
	}

	/**
	 * if a>＝b return true, else return false;
	 * 
	 * @param a
	 * @param b
	 * @return
	 */
	public static boolean compare(double a, double b) {
		if (a - b > -0.001)
			return true;
		else
			return false;
	}

	/**
	 * if a>b return true, else return false;
	 * 
	 * @param a
	 * @param b
	 * @return
	 */
	public static boolean bigger(double a, double b) {
		if (a - b > 0.001)
			return true;
		else
			return false;
	}

	/**
	 * 
	 * 返回不小于 value(保留scale位小数)的下一个数。 <br>
	 * 12.1310->12.14 <br>
	 * 12.13009->12.14
	 * 
	 * @param int scale 保留的小数位
	 * @return double
	 */
	public static double ceiling(double v, int scale) {

		if (scale < 0) {

			throw new IllegalArgumentException("The scale must be a positive integer or zero");

		}

		BigDecimal b = new BigDecimal(Double.toString(v));

		BigDecimal one = new BigDecimal("1");

		return b.divide(one, scale, BigDecimal.ROUND_UP).doubleValue();

	}

	/**
	 * 
	 * 返回不大于 value(保留scale位小数)的下一个数。
	 * 
	 * @param v
	 * @param scale
	 * @return
	 */
	public static double tailing(double v, int scale) {

		if (scale < 0) {
			throw new IllegalArgumentException("The scale must be a positive integer or zero");
		}

		BigDecimal b = new BigDecimal(Double.toString(v));

		BigDecimal one = new BigDecimal("1");

		return b.divide(one, scale, BigDecimal.ROUND_DOWN).doubleValue();

	}

	/**
	 * 
	 * 对精度后一位，进一取值 <br>
	 * 12.1310->12.14 <br>
	 * 12.1309->12.13
	 * 
	 * @param v
	 * @param scale
	 * @return
	 */
	public static double increaseOne(double v, int scale) {
		if (scale < 0) {
			throw new IllegalArgumentException("The scale must be a positive integer or zero");
		}

		BigDecimal b = new BigDecimal(Double.toString(v));

		return b.setScale(scale + 1, BigDecimal.ROUND_DOWN).setScale(scale, BigDecimal.ROUND_UP).doubleValue();
	}

}
```

测试
```java
public static void main(String[] args) {
	List<Integer> amountList = RedPackage.divideRedPackage(100, 10);
	for (Integer amount : amountList) {
		System.out.println("抢到金额：" + amount);
	}

	List<Double> amountList2 = RedPackage.divideRedPackage2(200, 30);
	for (Double amount : amountList2) {
		System.out.println("抢到金额：" + amount);
	}
}
```

### 3.2 写文件
```java
package com.xuzhihao.test;

import java.io.*;
import java.nio.file.Files;
import java.nio.file.Paths;

//字符流 Writer、Reader :FileWriter,BufferedWriter,PrintWriter
//字节流 InputStream、OutputStream :FileOutputStream,BufferedOutputStream,Files

public class WriteExample {
	public static void main(String[] args) throws IOException {
		// 构建写入内容
		StringBuilder stringBuilder = new StringBuilder();
		for (int i = 0; i < 1000000; i++) {
			stringBuilder.append("ABCDEFGHIGKLMNOPQRSEUVWXYZ");
		}
		// 写入内容
		final String content = stringBuilder.toString();
		// 存放文件的目录
		final String filepath1 = "D://write1.txt";
		final String filepath2 = "D://write2.txt";
		final String filepath3 = "D://write3.txt";
		final String filepath4 = "D://write4.txt";
		final String filepath5 = "D://write5.txt";
		final String filepath6 = "D://write6.txt";

		// 方法一:使用 FileWriter 写文件
		long stime1 = System.currentTimeMillis();
		fileWriterTest(filepath1, content);
		long etime1 = System.currentTimeMillis();
		System.out.println("FileWriter 写入用时:" + (etime1 - stime1));

		// 方法二:使用 BufferedWriter 写文件
		long stime2 = System.currentTimeMillis();
		bufferedWriterTest(filepath2, content);
		long etime2 = System.currentTimeMillis();
		System.out.println("BufferedWriter 写入用时:" + (etime2 - stime2));

		// 方法三:使用 PrintWriter 写文件
		long stime3 = System.currentTimeMillis();
		printWriterTest(filepath3, content);
		long etime3 = System.currentTimeMillis();
		System.out.println("PrintWriterTest 写入用时:" + (etime3 - stime3));

		// 方法四:使用 FileOutputStream 写文件
		long stime4 = System.currentTimeMillis();
		fileOutputStreamTest(filepath4, content);
		long etime4 = System.currentTimeMillis();
		System.out.println("FileOutputStream 写入用时:" + (etime4 - stime4));

		// 方法五:使用 BufferedOutputStream 写文件
		long stime5 = System.currentTimeMillis();
		bufferedOutputStreamTest(filepath5, content);
		long etime5 = System.currentTimeMillis();
		System.out.println("BufferedOutputStream 写入用时:" + (etime5 - stime5));

		// 方法六:使用 Files 写文件
		long stime6 = System.currentTimeMillis();
		filesTest(filepath6, content);
		long etime6 = System.currentTimeMillis();
		System.out.println("Files 写入用时:" + (etime6 - stime6));

	}

	/**
	 * 方法六:使用 Files 写文件
	 * 
	 * @param filepath 文件目录
	 * @param content  待写入内容
	 * @throws IOException
	 */
	private static void filesTest(String filepath, String content) throws IOException {
		Files.write(Paths.get(filepath), content.getBytes());
	}

	/**
	 * 方法五:使用 BufferedOutputStream 写文件
	 * 
	 * @param filepath 文件目录
	 * @param content  待写入内容
	 * @throws IOException
	 */
	private static void bufferedOutputStreamTest(String filepath, String content) throws IOException {
		try (BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(new FileOutputStream(filepath))) {
			bufferedOutputStream.write(content.getBytes());
		}
	}

	/**
	 * 方法四:使用 FileOutputStream 写文件
	 * 
	 * @param filepath 文件目录
	 * @param content  待写入内容
	 * @throws IOException
	 */
	private static void fileOutputStreamTest(String filepath, String content) throws IOException {
		try (FileOutputStream fileOutputStream = new FileOutputStream(filepath)) {
			byte[] bytes = content.getBytes();
			fileOutputStream.write(bytes);
		}
	}

	/**
	 * 方法三:使用 PrintWriter 写文件
	 * 
	 * @param filepath 文件目录
	 * @param content  待写入内容
	 * @throws IOException
	 */
	private static void printWriterTest(String filepath, String content) throws IOException {
		try (PrintWriter printWriter = new PrintWriter(new FileWriter(filepath))) {
			printWriter.print(content);
		}
	}

	/**
	 * 方法二:使用 BufferedWriter 写文件
	 * 
	 * @param filepath 文件目录
	 * @param content  待写入内容
	 * @throws IOException
	 */
	private static void bufferedWriterTest(String filepath, String content) throws IOException {
		try (BufferedWriter bufferedWriter = new BufferedWriter(new FileWriter(filepath))) {
			bufferedWriter.write(content);
		}
	}

	/**
	 * 方法一:使用 FileWriter 写文件
	 * 
	 * @param filepath 文件目录
	 * @param content  待写入内容
	 * @throws IOException
	 */
	private static void fileWriterTest(String filepath, String content) throws IOException {
		try (FileWriter fileWriter = new FileWriter(filepath)) {
			fileWriter.append(content);
		}
	}
}
```

### 3.3 hutool

#### 3.3.1 Convert
类型转换工具类，用于各种类型数据的转换。

```java 
//转换为字符串
int a = 1;
String aStr = Convert.toStr(a);
//转换为指定类型数组
String[] b = {"1", "2", "3", "4"};
Integer[] bArr = Convert.toIntArray(b);
//转换为日期对象
String dateStr = "2017-05-06";
Date date = Convert.toDate(dateStr);
//转换为列表
String[] strArr = {"a", "b", "c", "d"};
List<String> strList = Convert.toList(String.class, strArr);
```

#### 3.3.2 DateUtil
日期时间工具类，定义了一些常用的日期时间操作方法。

```java 
//Date、long、Calendar之间的相互转换
//当前时间
Date date = DateUtil.date();
//Calendar转Date
date = DateUtil.date(Calendar.getInstance());
//时间戳转Date
date = DateUtil.date(System.currentTimeMillis());
//自动识别格式转换
String dateStr = "2017-03-01";
date = DateUtil.parse(dateStr);
//自定义格式化转换
date = DateUtil.parse(dateStr, "yyyy-MM-dd");
//格式化输出日期
String format = DateUtil.format(date, "yyyy-MM-dd");
//获得年的部分
int year = DateUtil.year(date);
//获得月份，从0开始计数
int month = DateUtil.month(date);
//获取某天的开始、结束时间
Date beginOfDay = DateUtil.beginOfDay(date);
Date endOfDay = DateUtil.endOfDay(date);
//计算偏移后的日期时间
Date newDate = DateUtil.offset(date, DateField.DAY_OF_MONTH, 2);
//计算日期时间之间的偏移量
long betweenDay = DateUtil.between(date, newDate, DateUnit.DAY);
```

#### 3.3.3 StrUtil
字符串工具类，定义了一些常用的字符串操作方法。

```java 
//判断是否为空字符串
String str = "test";
StrUtil.isEmpty(str);
StrUtil.isNotEmpty(str);
//去除字符串的前后缀
StrUtil.removeSuffix("a.jpg", ".jpg");
StrUtil.removePrefix("a.jpg", "a.");
//格式化字符串
String template = "这只是个占位符:{}";
String str2 = StrUtil.format(template, "我是占位符");
LOGGER.info("/strUtil format:{}", str2);
```

#### 3.3.4 ClassPathResource

获取classPath下的文件，在Tomcat等容器下，classPath一般是WEB-INF/classes。

```java 
//获取定义在src/main/resources文件夹中的配置文件
ClassPathResource resource = new ClassPathResource("generator.properties");
Properties properties = new Properties();
properties.load(resource.getStream());
LOGGER.info("/classPath:{}", properties);
```

#### 3.3.5 ReflectUtil
Java反射工具类，可用于反射获取类的方法及创建对象。

```java 
//获取某个类的所有方法
Method[] methods = ReflectUtil.getMethods(PmsBrand.class);
//获取某个类的指定方法
Method method = ReflectUtil.getMethod(PmsBrand.class, "getId");
//使用反射来创建对象
PmsBrand pmsBrand = ReflectUtil.newInstance(PmsBrand.class);
//反射执行对象的方法
ReflectUtil.invoke(pmsBrand, "setId", 1);
```

#### 3.3.6 NumberUtil
数字处理工具类，可用于各种类型数字的加减乘除操作及判断类型。

```java 
double n1 = 1.234;
double n2 = 1.234;
double result;
//对float、double、BigDecimal做加减乘除操作
result = NumberUtil.add(n1, n2);
result = NumberUtil.sub(n1, n2);
result = NumberUtil.mul(n1, n2);
result = NumberUtil.div(n1, n2);
//保留两位小数
BigDecimal roundNum = NumberUtil.round(n1, 2);
String n3 = "1.234";
//判断是否为数字、整数、浮点数
NumberUtil.isNumber(n3);
NumberUtil.isInteger(n3);
NumberUtil.isDouble(n3);
```

#### 3.3.7 BeanUtil

JavaBean的工具类，可用于Map与JavaBean对象的互相转换以及对象属性的拷贝。

```java 
PmsBrand brand = new PmsBrand();
brand.setId(1L);
brand.setName("小米");
brand.setShowStatus(0);
//Bean转Map
Map<String, Object> map = BeanUtil.beanToMap(brand);
LOGGER.info("beanUtil bean to map:{}", map);
//Map转Bean
PmsBrand mapBrand = BeanUtil.mapToBean(map, PmsBrand.class, false);
LOGGER.info("beanUtil map to bean:{}", mapBrand);
//Bean属性拷贝
PmsBrand copyBrand = new PmsBrand();
BeanUtil.copyProperties(brand, copyBrand);
LOGGER.info("beanUtil copy properties:{}", copyBrand);
```

#### 3.3.8 CollUtil
集合操作的工具类，定义了一些常用的集合操作。

```java 
//数组转换为列表
String[] array = new String[]{"a", "b", "c", "d", "e"};
List<String> list = CollUtil.newArrayList(array);
//join：数组转字符串时添加连接符号
String joinStr = CollUtil.join(list, ",");
LOGGER.info("collUtil join:{}", joinStr);
//将以连接符号分隔的字符串再转换为列表
List<String> splitList = StrUtil.split(joinStr, ',');
LOGGER.info("collUtil split:{}", splitList);
//创建新的Map、Set、List
HashMap<Object, Object> newMap = CollUtil.newHashMap();
HashSet<Object> newHashSet = CollUtil.newHashSet();
ArrayList<Object> newList = CollUtil.newArrayList();
//判断列表是否为空
CollUtil.isEmpty(list);
```

#### 3.3.9 MapUtil
Map操作工具类，可用于创建Map对象及判断Map是否为空。

```java 
//将多个键值对加入到Map中
Map<Object, Object> map = MapUtil.of(new String[][]{
    {"key1", "value1"},
    {"key2", "value2"},
    {"key3", "value3"}
});
//判断Map是否为空
MapUtil.isEmpty(map);
MapUtil.isNotEmpty(map);
```

#### 3.3.10 AnnotationUtil

注解工具类，可用于获取注解与注解中指定的值。

```java 
//获取指定类、方法、字段、构造器上的注解列表
Annotation[] annotationList = AnnotationUtil.getAnnotations(HutoolController.class, false);
LOGGER.info("annotationUtil annotations:{}", annotationList);
//获取指定类型注解
Api api = AnnotationUtil.getAnnotation(HutoolController.class, Api.class);
LOGGER.info("annotationUtil api value:{}", api.description());
//获取指定类型注解的值
Object annotationValue = AnnotationUtil.getAnnotationValue(HutoolController.class, RequestMapping.class);
```

#### 3.3.11 SecureUtil
加密解密工具类，可用于MD5加密。

```java 
//MD5加密
String str = "123456";
String md5Str = SecureUtil.md5(str);
LOGGER.info("secureUtil md5:{}", md5Str);
```

#### 3.3.12 CaptchaUtil
验证码工具类，可用于生成图形验证码。

```java 
//生成验证码图片
LineCaptcha lineCaptcha = CaptchaUtil.createLineCaptcha(200, 100);
try {
    request.getSession().setAttribute("CAPTCHA_KEY", lineCaptcha.getCode());
    response.setContentType("image/png");//告诉浏览器输出内容为图片
    response.setHeader("Pragma", "No-cache");//禁止浏览器缓存
    response.setHeader("Cache-Control", "no-cache");
    response.setDateHeader("Expire", 0);
    lineCaptcha.write(response.getOutputStream());
} catch (IOException e) {
    e.printStackTrace();
}
```

#### 3.3.13 其他工具类
- hutool-aop JDK 动态代理封装，提供非 IOC 下的切面支持
- hutool-bloomFilter 布隆过滤，提供一些 Hash 算法的布隆过滤
- hutool-cache 缓存
- hutool-core 核心，包括 Bean 操作、日期、各种 Util 等
- hutool-cron 定时任务模块，提供类 Crontab 表达式的定时任务
- hutool-crypto 加密解密模块
- hutool-db JDBC 封装后的数据操作，基于 ActiveRecord 思想
- hutool-dfa 基于 DFA 模型的多关键字查找
- hutool-extra 扩展模块，对第三方封装（模板引擎、邮件等）
- hutool-http 基于 HttpUrlConnection 的 Http 客户端封装
- hutool-log 自动识别日志实现的日志门面
- hutool-script 脚本执行封装，例如 Javascript
- hutool-setting 功能更强大的 Setting 配置文件和 Properties 封装
- hutool-system 系统参数调用封装（JVM 信息等）
- hutool-json JSON 实现
- hutool-captcha 图片验证码实现

Hutool中的工具类很多，可以参考：[https://www.hutool.cn/](https://www.hutool.cn/)

### 3.4 JDBC枚举单例

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class MyObject {
    private static final String driver = "com.mysql.cj.jdbc.Driver";// mysql驱动
    // private static final String driver = "oracle.jdbc.driver.OracleDriver"; // oracle驱动
    // private static final String driver = "org.postgresql.Driver"; // postgresql驱动
    // private static final String driver = "com.microsoft.sqlserver.jdbc.SQLServerDriver"; // sqlserver驱动

    private static final String url = "jdbc:mysql://debug-registry:3306/mall";// mysql地址
    // private static final String url = "jdbc:oracle:thin:@debug-registry:1521:ORCL"; // oracle地址
    // private static final String url = "jdbc:postgresql://debug-registry:5432/VJSP10003182"; // postgresql地址
    // private static final String url = "jdbc:sqlserver://debug-registry:1433; DatabaseName=maptest"; // sqlserver地址

    public enum JdbcUtilSingletion {
        connectionFactory;

        private Connection connection;

        private JdbcUtilSingletion() {
            try {
                String username = "root";
                String password = "root";
                Class.forName(driver);
                connection = DriverManager.getConnection(url, username, password);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        public Connection getConnection() {
            return connection;
        }
    }

    public static Connection getConnection() {
        return JdbcUtilSingletion.connectionFactory.getConnection();
    }

    public static void main(String[] args) {
        MyThread t1 = new MyThread();
        MyThread t2 = new MyThread();
        MyThread t3 = new MyThread();
        t1.start();
        t2.start();
        t3.start();
    }

    public static void main2(String[] args) throws SQLException {

        // 首先需要获取实例,然后获取连接
        Connection conn = MyObject.getConnection();
        // 创建语句
        Statement st = conn.createStatement();
        // 执行语句
        ResultSet rs = st.executeQuery("select * from ums_admin");
        // 处理结果集
        while (rs.next()) {
            System.out
                .println(rs.getObject(1) + "\t" + rs.getObject(2) + "\t" + rs.getObject(3) + "\t" + rs.getObject(4));
        }
        st.executeBatch();
        st.close();

        // 批量插入
        String sql = "INSERT INTO ums_admin2 (username,password) VALUES(?,?)";
        PreparedStatement pstmt = conn.prepareStatement(sql);
        for (int i = 1; i <= 10; i++) {
            pstmt.setString(1, "username" + i);
            pstmt.setString(2, "password" + i);
            pstmt.addBatch();
        }
        pstmt.executeBatch();
        pstmt.close();
        conn.close();
    }
}
```

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        super.run();
        for (int i = 0; i < 5; i++) {
            System.out.println(MyObject.getConnection().hashCode());
        }
    }
}
```