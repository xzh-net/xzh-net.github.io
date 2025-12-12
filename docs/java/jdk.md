
# JDK特性

## 1. 语法糖

### 1.1 默认构造器
```java
```
### 1.2 自动拆装箱 JDK 5 
```java
```
### 1.3 泛型集合取值 JDK 5 
```java
```
### 1.5 可变参数 JDK 5 

jdbc释放链接

```java
//释放资源
public static void free(AutoCloseable... ios) {
		for (AutoCloseable io : ios) {
			try {
				if (io != null) {
					io.close();
				}
			} catch (Exception e) {
				e.printStackTrace();
			} finally {
				try {
					if (io != null) {
						io.close();
					}
				} catch (Exception e) {
					e.printStackTrace();
				} finally {
					if (io != null) {
						try {
							io.close();
						} catch (Exception e) {
							e.printStackTrace();
						}
					}
				}
			}
		}
	}
```
### 1.6 foreach 循环 JDK 5

能够配合数组，以及所有实现了 Iterable 接口的集合类一起使用
```java
```
### 1.7 switch 字符串 JDK 7 
```java
```
### 1.8 switch 枚举 JDK 7 
```java
```
### 1.9 枚举类
```java
```
### 1.10 try-with-resources JDK 7

addSuppressed异常压制
```java
```
### 1.11 方法重写时的桥接方法 子类返回值可以是父类返回值的子类
```java
```
### 1.12 匿名内部类
```java
```

## 2. JDK8

### 2.1 Stream使用和底层原理详解
### 2.2 函数式接口与Lambda 表达式详解
### 2.3 方法引用与Data API详解
### 2.4 其他特性详解

## 3. JDK9

### 3.1 modularity System 模块化系统
### 3.1 HTTP/2详解
### 3.2 JShell : 交互式 Java REPL
### 3.3 不可变集合工厂方法、私有接口方法详解
### 3.4 多版本兼容 JAR与I/O 流新特性

## 4. JDK10

### 4.1 局部变量的类型推断
### 4.2 GC改进和内存管理
### 4.3 线程本地握手
### 4.4 备用内存设备上的堆分配
### 4.5 其他Unicode语言 - 标记扩展
### 4.6 基于Java的实验性JIT编译器
### 4.7 开源根证书
### 4.8 根证书颁发认证（CA）
### 4.9 将JDK生态整合单个存储库
### 4.10 删除工具javah

## 5. JDK11

### 5.1 HTTP Client加强
### 5.2 增强局部变量类型推断var
### 5.3 Java Flight Recorder (JFR)
### 5.4 JavaFX从JDK分离为独立模块
### 5.5 Project ZGC
### 5.6 完全支持Linux容器（包括Docker）

## 6. JDK12

### 6.1 Shenandoah
### 6.2 Microbenchmark Suite
### 6.3 JVM常量API
### 6.4 默认CDS档案
### 6.5 G1的优化详解

## 7. JDK13

### 7.1 Dynamic CDS Archives
### 7.2 ZGC详解
### 7.3 Reimplement the Legacy Socket API
### 7.4 Switch Expressions
### 7.5 Text Blocks