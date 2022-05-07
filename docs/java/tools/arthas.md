# Arthas 简介

Arthas是Alibaba开源的Java诊断工具，深受开发者喜爱。它采用命令行交互模式，同时提供丰富的 Tab 自动补全功能，进一步方便进行问题的定位和诊断

```
wget https://alibaba.github.io/arthas/arthas-boot.jar
java -jar arthas-boot.jar
```

在线安装
```
curl -L https://alibaba.github.io/arthas/install.sh | sh
```

## 1. 常用命令

### 1.1 dashboard

使用`dashboard`命令可以显示当前系统的实时数据面板，包括线程信息、JVM内存信息及JVM运行时参数。

![](../../assets/_images/java/tools/arthas/arthas_start_03.png)

### 1.2 thread

查看当前线程信息，查看线程的堆栈，可以找出当前最占CPU的线程。

![](../../assets/_images/java/tools/arthas/arthas_start_04.png)

常用命令：

```bash
# 打印当前最忙的3个线程的堆栈信息
thread -n 3
# 指定采样时间间隔
thread -n 3 -i 10000
# 查看ID为1都线程的堆栈信息
thread 1
# 找出当前阻塞其他线程的线程
thread -b
# 查看指定状态的线程
thread -state WAITING
```

### 1.3 logger

使用`logger`命令可以查看日志信息，并改变日志级别，这个命令非常有用。

比如我们在生产环境上一般是不会打印`DEBUG`级别的日志的，当我们在线上排查问题时可以临时开启`DEBUG`级别的日志，帮助我们排查问题，下面介绍下如何操作。

- 我们的应用默认使用的是`INFO`级别的日志，使用`logger`命令可以查看；

![](../../assets/_images/java/tools/arthas/arthas_start_06.png)

- 使用如下命令改变日志级别为`DEBUG`，需要使用`-c`参数指定类加载器的HASH值；

```bash
logger -c 21b8d17c --name ROOT --level debug
```

- 再使用`logger`命令查看，发现`ROOT`级别日志已经更改；

![](../../assets/_images/java/tools/arthas/arthas_start_07.png)

- 使用`docker logs -f shop-admin`命令查看容器日志，发现已经打印了DEBUG级别的日志；

![](../../assets/_images/java/tools/arthas/arthas_start_08.png)

- 查看完日志以后记得要把日志级别再调回`INFO`级别。

```bash
logger -c 21b8d17c --name ROOT --level info
```

### 1.4 热更新

```bash
jad --source-only com.xuzhihao.LoginAction > /tmp/LoginAction.java
sc -d *LoginAction | grep classLoaderHash
mc -c 50696e43 /tmp/LoginAction.java -d  /tmp
redefine /tmp/com/xuzhihao/LoginAction.class
```

## 2. 命令列表

### 2.1 基础命令

```
help——查看命令帮助信息
cls——清空当前屏幕区域
session——查看当前会话的信息
reset——重置增强类，将被 Arthas 增强过的类全部还原，Arthas 服务端关闭时会重置所有增强过的类
version——输出当前目标 Java 进程所加载的 Arthas 版本号
history——打印命令历史
quit——退出当前 Arthas 客户端，其他 Arthas 客户端不受影响
shutdown——关闭 Arthas 服务端，所有 Arthas 客户端全部退出
keymap——Arthas快捷键列表及自定义快捷键
```

### 2.2 Jvm相关

```
dashboard——当前系统的实时数据面板
thread——查看当前 JVM 的线程堆栈信息
jvm——查看当前 JVM 的信息
sysprop——查看和修改JVM的系统属性
sysenv——查看JVM的环境变量
getstatic——查看类的静态属性
ognl——执行ognl表达式
mbean——查看 Mbean 的信息
```

### 2.3 class/classloader相关

```
sc——查看JVM已加载的类信息
sm——查看已加载类的方法信息
jad——反编译指定已加载类的源码
mc——内存编绎器，内存编绎.java文件为.class文件
redefine——加载外部的.class文件，redefine到JVM里
dump——dump 已加载类的 byte code 到特定目录
classloader——查看classloader的继承树，urls，类加载信息，使用classloader去getResource
```

### 2.4 monitor/watch/trace相关

请注意，这些命令，都通过字节码增强技术来实现的，会在指定类的方法中插入一些切面来实现数据统计和观测，因此在线上、预发使用时，请尽量明确需要观测的类、方法以及条件，诊断结束要执行 shutdown 或将增强过的类执行 reset 命令。

```
monitor——方法执行监控
watch——方法执行数据观测
trace——方法内部调用路径，并输出方法路径上的每个节点上耗时
stack——输出当前方法被调用的调用路径
tt——方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测
```

### 2.5 options

```
options——查看或设置Arthas全局开关
```