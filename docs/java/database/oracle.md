# Oracle

## 1. 实例

### 1.1 物理存储结构

- Data File 数据文件
- Control File 控制文件
- Redo Log File 重做日志文件

![](../../assets/_images/java/database/oracle/2.png)


```sql
col name for a50
SELECT name FROM v$datafile
union all
SELECT name FROM v$controlfile
union all
SELECT member FROM v$lofile；
```

查看分配给数据库的内存大小

```sql
show parameter memory 
```

### 1.2 内存结构

![](../../assets/_images/java/database/oracle/1.png)


是一种数据库访问机制，主要由内存结构和进程结构组成

内存结构主要包括系统全局区（System Global Area，`SGA`）、进程全局区（Process Global Area，`PGA`）等


### 1.3 Database Buffer Cache的组成

Buffer Cache 有两个管理列表：写列表和最近最少使⽤的(LRU)列表。

Buffer Cache中的内存缓冲区分成三部分：
1. free buffer 空闲缓冲区不包含任何有⽤的数据，数据库可以重⽤他们保存从磁盘读取的新数据
2. dirty buffer 脏缓冲区，包含已修改但尚未写到磁盘的数据。
3. pinned buffer 钉住/保留缓冲区，是⽤户会话当前正在激活使⽤的数据缓冲区

### 1.4 Database Buffer Cache

LRU 缓冲区清除算法

### 1.5 Shared Pool

### 1.6 SQL在Oracle内部的处理流程

1. 软解析
   - 任何不适硬解析的解析都是软解析。如果提交的语句与在共享式中某个可重⽤SQL语句相同，则数据库将重⽤该现有代码。重⽤代码也称为库缓存命中。⼀般的，软解析⽐硬解析更可取，因为数据库可以跳过优化和⾏源⽣成步骤，⽽直接进⼊到直⾏阶段。下图是在专⽤服务器体系结构中，⼀个update语句的共享池检查的简化表示
2. 硬解析
   - 如果数据库不能重⽤现有代码，则它必须⽣成应⽤程序代码的⼀个新的可执⾏版本，此操作称为⼀个硬解析，或库缓存未命中。数据库对DDL始终执⾏硬解析。
   在硬解析期间，数据库多次访问库缓存和数据字典缓存以检查数据字典。当数据库访问这些区域时，它在所需对象上使⽤⼀个叫做闩锁的串⾏化设备，以便它们的定义不糊被更改。闩锁的争⽤会增加语句的执⾏时间，并降低并发

### 1.7 系统五大后台常驻进程
- 数据写入进程（DBWR）
- 检查点进程（CKPT）
- 日志写入进程（LGWR）
- 系统监控进程（SMON）
- 进程监控进程（PMON）

netstat -tulnp|grep 5560 -查看端口号是否被占用


## 2. 事务隔离级别

| 隔离级别| 脏读可能性 | 不可重复度可能性 | 幻读可能性 | 加锁读 |
| ----- | ----- | ----- | ----- | ----- |
| READ UNCOMMITTED| 是 | 是 | 是 | 否 |
| READ COMMITTED| 否 | 是 | 是 | 否 |
| REPEATABLE READ| 否 | 否 | 是 | 否 |
| SERIALIZABLE| 否 | 否 | 否 | 是 |

## 3. MySQL & Oracle 区别

1、类型和成本的区别

oracle数据库是一个对象关系数据库管理系统（ORDBMS），一个重量型数据库。它通常被称为Oracle RDBMS或简称为Oracle，是一个收费的数据库。

MySQL是一个开源的关系数据库管理系统（RDBMS），一个是轻量型数据库。它是世界上使用最多的RDBMS，作为服务器运行，提供对多个数据库的多用户访问。它是一个开源、免费的数据库。

2、存储上的区别

与Oracle相比，MySQL没有表空间，角色管理，快照，同义词和包以及自动存储管理。

3、安全性上的区别

MySQL使用三个参数来验证用户，即用户名，密码和位置；Oracle使用了许多安全功能，如用户名，密码，配置文件，本地身份验证，外部身份验证，高级安全增强功能等。

4、对事务的支持

MySQL在innodb存储引擎的行级锁的情况下才可支持事务，而Oracle则完全支持事务

5、性能诊断上的区别

MySQL的诊断调优方法较少，主要有慢查询日志。

Oracle有各种成熟的性能诊断调优工具，能实现很多自动分析、诊断功能。比如awr、addm、sqltrace、tkproof等

6、管理工具上的区别

MySQL管理工具较少，在linux下的管理工具的安装有时要安装额外的包（phpmyadmin， etc)，有一定复杂性。

Oracle有多种成熟的命令行、图形界面、web管理工具，还有很多第三方的管理工具，管理极其方便高效。

7、并发性上的区别

MySQL以表级锁为主，对资源锁定的粒度很大，如果一个session对一个表加锁时间过长，会让其他session无法更新此表中的数据。虽然InnoDB引擎的表可以用行级锁，但这个行级锁的机制依赖于表的索引，如果表没有索引，或者sql语句没有使用索引，那么仍然使用表级锁。

Oracle使用行级锁，对资源锁定的粒度要小很多，只是锁定sql需要的资源，并且加锁是在数据库中的数据行上，不依赖与索引。所以Oracle对并发性的支持要好很多。

8、 保存数据的持久性

MySQL是在数据库更新或者重启，则会丢失数据，Oracle把提交的sql操作线写入了在线联机日志文件中，保持到了磁盘上，可以随时恢复

9、事务隔离级别上的区别

MySQL是read commited的隔离级别，而Oracle是repeatable read的隔离级别，同时二者都支持serializable串行化事务隔离级别，可以实现最高级别的读一致性。每个session提交后其他session才能看到提交的更改。

Oracle通过在undo表空间中构造多版本数据块来实现读一致性，每个session查询时，如果对应的数据块发生变化，Oracle会在undo表空间中为这个session构造它查询时的旧的数据块

MySQL没有类似Oracle的构造多版本数据块的机制，只支持read commited的隔离级别。一个session读取数据时，其他session不能更改数据，但可以在表最后插入数据。session更新数据时，要加上排它锁，其他session无法访问数据。

10、操作上的一些区别

①主键

Mysql一般使用自动增长类型，在创建表时只要指定表的主键为auto_increment，插入记录时，不需要再指定该记录的主键值，Mysql将自动增长；

Oracle没有自动增长类型，主键一般使用的序列，插入记录时将序列号的下一个值付给该字段即可；只是ORM框架是只要是native主键生成策略即可。

②单引号的处理

MYSQL里可以用双引号包起字符串，ORACLE里只可以用单引号包起字符串。在插入和修改字符串前必须做单引号的替换：把所有出现的一个单引号替换成两个单引号。

③翻页的SQL语句的处理

MYSQL处理翻页的SQL语句比较简单，用LIMIT 开始位置，记录个数；ORACLE处理翻页的SQL语句就比较繁琐了。

④ 空字符的处理

MYSQL的非空字段也可以有空的内容，ORACLE里定义了非空字段就不容许有空的内容。

⑤字符串的模糊比较

MYSQL里用 字段名 like '%字符串%'；ORACLE里也可以用 字段名 like '%字符串%' 但这种方法不能使用索引， 速度不快。


## 4. SQL范式

1. 第一范式:确保每列的原子性
2. 第二范式:在第一范式的基础上更进一层,目标是确保表中的每列都和主键相关
3. 第三范式:目标是确保每列都和主键列直接相关,而不是间接相关. 
