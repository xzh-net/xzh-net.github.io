# HBase 2.1.0

## 1. 安装

### 1.1 上传解压HBase安装包

```bash
mkdir -p /export/software /export/server
cd /export/software 
tar -xvzf hbase-2.1.0.tar.gz -C ../server/
```

### 1.2 修改HBase配置文件

#### 1.2.1 hbase-env.sh

```bash
cd /export/server/hbase-2.1.0/conf
vim hbase-env.sh
# 第28行
export JAVA_HOME=/export/server/jdk1.8.0_202/
export HBASE_MANAGES_ZK=false
```

#### 1.2.2 hbase-site.xml

vim hbase-site.xml

```xml
<configuration>
        <!-- HBase数据在HDFS中的存放的路径 -->
        <property>
            <name>hbase.rootdir</name>
            <value>hdfs://node1.xuzhihao.net:8020/hbase</value>
        </property>
        <!-- Hbase的运行模式。false是单机模式，true是分布式模式。若为false,Hbase和Zookeeper会运行在同一个JVM里面 -->
        <property>
            <name>hbase.cluster.distributed</name>
            <value>true</value>
        </property>
        <!-- ZooKeeper的地址 -->
        <property>
            <name>hbase.zookeeper.quorum</name>
            <value>node1.xuzhihao.net,node2.xuzhihao.net,node3.xuzhihao.net</value>
        </property>
        <!-- ZooKeeper快照的存储位置 -->
        <property>
            <name>hbase.zookeeper.property.dataDir</name>
            <value>/export/server/apache-zookeeper-3.6.0-bin/data</value>
        </property>
        <!--  V2.1版本，在分布式情况下, 设置为false -->
        <property>
            <name>hbase.unsafe.stream.capability.enforce</name>
            <value>false</value>
        </property>
</configuration>

```


### 1.3 配置环境变量

```bash
# 配置Hbase环境变量
vim /etc/profile
export HBASE_HOME=/export/server/hbase-2.1.0
export PATH=$PATH:${HBASE_HOME}/bin:${HBASE_HOME}/sbin

#加载环境变量
source /etc/profile
```

### 1.4 复制jar包到lib

```bash
cp $HBASE_HOME/lib/client-facing-thirdparty/htrace-core-3.1.0-incubating.jar $HBASE_HOME/lib/
```

### 1.5 修改regionservers文件

vim regionservers
```bash
node1.xuzhihao.net
node2.xuzhihao.net
node3.xuzhihao.net
```

### 1.6 分发安装包与配置文件

```bash
cd /export/server
scp -r hbase-2.1.0/ node2.xuzhihao.net:$PWD
scp -r hbase-2.1.0/ node3.xuzhihao.net:$PWD

scp -r /etc/profile node2.xuzhihao.net:/etc
scp -r /etc/profile node3.xuzhihao.net:/etc

#在node2.xuzhihao.net和node3.xuzhihao.net加载环境变量
source /etc/profile
```

### 1.7 启动HBase

```bash
cd /export/onekey
# 启动ZK
./start-zk.sh
# 启动hadoop
start-dfs.sh
# 启动hbase
start-hbase.sh
```

### 1.8 验证Hbase是否启动成功

```bash
# 启动hbase shell客户端
hbase shell
# 输入status
```

### 1.9 WebUI

http://node1.xuzhihao.net:16010/master-status


### 1.10 参考硬件配置

| 进程 | 堆 | 描述 |
| :---------- | :---------- | :---------------------------------- |
| NameNode    | 8 GB | 每100TB数据或每100W个文件大约占用NameNode堆1GB的内存 |
| SecondaryNameNode | 8 GB | 在内存中重做主NameNode的EditLog，因此配置需要与NameNode一样 |
| DataNode | 1 GB | 适度即可 |
| ResourceManager | 4 GB | 适度即可（注意此处是MapReduce的推荐配置） |
| NodeManager | 2 GB | 适当即可（注意此处是MapReduce的推荐配置） |
| HBase HMaster | 4 GB | 轻量级负载，适当即可 |
| HBase RegionServer | 12 GB | 大部分可用内存、同时为操作系统缓存、任务进程留下足够的空间 |
| ZooKeeper | 1 GB | 适度 |


推荐：
- Master机器要运行NameNode、ResourceManager、以及HBase HMaster，推荐24GB左右
- Slave机器需要运行DataNode、NodeManager和HBase RegionServer，推荐24GB（及以上）

## 2. 基本命令

```shell
# 表操作
create "ORDER_INFO", 'C1', 'C2'      # 创建订单表ORDER_INFO，有一个列蔟为C1，C2
create "MOMO_CHAT:MSG", "C1"         # 指定命名空间创建表
alter 'ORDER_INFO', 'C3'             # 新增列蔟C3
alter 'ORDER_INFO', 'delete' => 'C3' # 删除列簇 C3

alter "MOMO_CHAT:MSG", {NAME => "C1", COMPRESSION => "GZ"}  # 指定修改某个表的列蔟，它的压缩方式
create 'MOMO_CHAT:MSG', {NAME => "C1", COMPRESSION => "GZ"}, { NUMREGIONS => 6, SPLITALGO => 'HexStringSplit'} # 在创建表时需要指定预分区

create_namespace 'MOMO_CHAT'         # 创建一个命名空间
list_namespace                       # 查看命名空间
drop_namespace 'MOMO_CHAT'           # 删除之前的命名空间
describe_namespace 'MOMO_CHAT'       # 查看某个具体的命名空间
describe_namespace 'default'

disable "ORDER_INFO"                 # 禁用表
drop "ORDER_INFO"                    # 删除表
disable "MOMO_CHAT:MSG" 
drop "MOMO_CHAT:MSG"

# 添加数据 put '表名','ROWKEY','列蔟名:列名','值'
put "ORDER_INFO", "000001", "C1:STATUS", "已提交"
put "ORDER_INFO", "000001", "C1:PAY_MONEY", 4070
put "ORDER_INFO", "000001", "C1:PAYWAY", 1
put "ORDER_INFO", "000001", "C1:USER_ID", "4944191"
put "ORDER_INFO", "000001", "C1:OPERATION_DATE", "2020-04-25 12:09:16"
put "ORDER_INFO", "000001", "C1:CATEGORY", "手机;"

# 查询操作
get "ORDER_INFO", "000001" # 查询ORDER_INFO rowkey为：000001
get "ORDER_INFO", "000001", {FORMATTER => 'toString'}   # 指定字符集
put "ORDER_INFO", "000001", "C1:STATUS", "已付款" # 更新状态

# 删除操作
delete "ORDER_INFO", "000001", "C1:STATUS" # 删除列
deleteall "ORDER_INFO", "000001"    # 删除所有列

count "ORDER_INFO"  # 查询记录数

# scan操作
scan "ORDER_INFO", {FORMATTER => 'toString'}    # 查询订单所有数据
scan "ORDER_INFO", {FORMATTER => 'toString', LIMIT => 3} # 查询订单数据（只显示3条）
scan "ORDER_INFO", {FORMATTER => 'toString', LIMIT => 3, COLUMNS => ['C1:STATUS', 'C1:PAYWAY']} # 显示指定列 只显示3条

# scan '表名', {ROWPREFIXFILTER => 'rowkey'}
scan "ORDER_INFO", {ROWPREFIXFILTER => '02602f66-adc7-40d4-8485-76b5632b5b53',FORMATTER => 'toString', LIMIT => 3, COLUMNS => ['C1:STATUS', 'C1:PAYWAY']}

# Scan + Filter 只查询订单的ID为：02602f66-adc7-40d4-8485-76b5632b5b53、订单状态以及支付方式
scan "ORDER_INFO", {FILTER => "RowFilter(=,'binary:02602f66-adc7-40d4-8485-76b5632b5b53')", COLUMNS => ['C1:STATUS', 'C1:PAYWAY'], FORMATTER => 'toString'}

# 查询状态为「已付款」的订单
scan "ORDER_INFO", {FILTER => "SingleColumnValueFilter('C1', 'STATUS', =, 'binary:已付款')", FORMATTER => 'toString'}

# 查询支付方式为1，且金额大于3000的订单
scan "ORDER_INFO", {FILTER => "SingleColumnValueFilter('C1', 'PAYWAY', =, 'binary:1') AND SingleColumnValueFilter('C1', 'PAY_MONEY', >, 'binary:3000')", FORMATTER => 'toString'}

# HBase的计数器 对0000000020新闻01:00 - 02:00访问计数+1
get_counter 'NEWS_VISIT_CNT','0000000020_01:00-02:00', 'C1:CNT'
incr 'NEWS_VISIT_CNT','0000000020_01:00-02:00','C1:CNT'
```

## 3. Phoenix

### 3.1 简介

* Apache Phoenix基于HBase的一个SQL引擎，我们可以使用Phoenix在HBase之上提供SQL语言的支持。
* Phoenix是可以支持二级索引的，而且Phoenix它自动帮助我们管理二级索引，底层是通过HBase的协处理器来实现的，通过配合二级索引和HBase rowkey，可以提升hbase的查询效率
* Phoenix底层还是将SQL语言解析为HBase的原生查询（put/get/scan），所以它的定位还是在随机实时查询——OLTP领域
* Apache Phoenix不是独立运行的，而是提供一些JAR包，扩展了HBase的功能

### 3.2 Phoneix索引的分类

#### 3.2.1 全局索引

* 全局索引适用于读多写少业务
* 全局索引绝大多数负载都发生在写入时，当构建了全局索引时，Phoenix会拦截写入(DELETE、UPSERT值和UPSERT SELECT)上的数据表更新，构建索引更新，同时更新所有相关的索引表，开销较大
* 读取时，Phoenix将选择最快能够查询出数据的索引表。默认情况下，除非使用Hint，如果SELECT查询中引用了其他非索引列，该索引是不会生效的
* 全局索引一般和覆盖索引搭配使用，读的效率很高，但写入效率会受影响

#### 3.2.2 本地索引

* 本地索引适合写操作频繁，读相对少的业务
* 当使用SQL查询数据时，Phoenix会自动选择是否使用本地索引查询数据
* 在本地索引中，索引数据和业务表数据存储在同一个服务器上，避免写入期间的其他网络开销
* 在Phoenix 4.8.0之前，本地索引保存在一个单独的表中，在Phoenix 4.8.1中，本地索引的数据是保存在一个影子列蔟中
* 本地索引查询即使SELECT引用了非索引中的字段，也会自动应用索引的

注意：创建表的时候指定了SALT_BUCKETS，是不支持本地索引的


#### 3.2.3 覆盖索引

Phoenix提供了覆盖的索引，可以不需要在找到索引条目后返回到主表。Phoenix可以将关心的数据捆绑在索引行中，从而节省了读取时间的开销

#### 3.2.4 函数索引

函数索引(4.3和更高版本)可以支持在列上创建索引，还可以基于任意表达式上创建索引。然后，当查询使用该表达式时，可以使用索引来检索结果，而不是数据表。例如，可以在UPPER(FIRST_NAME||‘ ’||LAST_NAME)上创建一个索引，这样将来搜索两个名字拼接在一起时，索引依然可以生效

### 3.3 安装

http://phoenix.apache.org/download.html

```bash
# 1.上传安装包到Linux系统，并解压
cd /export/software
tar -xvzf apache-phoenix-5.0.0-HBase-2.0-bin.tar.gz -C ../server/

# 2.将phoenix的所有jar包添加到所有HBase RegionServer和Master的复制到HBase的lib目录
cp /export/server/apache-phoenix-5.0.0-HBase-2.0-bin/phoenix-*.jar /export/server/hbase-2.1.0/lib/  # 拷贝jar包到hbase lib目录 
cd /export/server/hbase-2.1.0/lib/  # 进入到hbase lib目录
# 分发jar包到每个HBase 节点
scp phoenix-*.jar node2.xuzhihao.net:$PWD
scp phoenix-*.jar node3.xuzhihao.net:$PWD

# 3.修改配置文件
cd /export/server/hbase-2.1.0/conf/
vim hbase-site.xml
```

```xml
<!-- 将以下配置添加到 hbase-site.xml 后边 -->
<!-- 支持HBase命名空间映射 -->
<property>
    <name>phoenix.schema.isNamespaceMappingEnabled</name>
    <value>true</value>
</property>
<!-- 支持索引预写日志编码 -->
<property>
  <name>hbase.regionserver.wal.codec</name>
  <value>org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec</value>
</property>
```

```bash
# 将hbase-site.xml分发到每个节点
scp hbase-site.xml node2.xuzhihao.net:$PWD
scp hbase-site.xml node3.xuzhihao.net:$PWD

# 4. 将配置后的hbase-site.xml拷贝到phoenix的bin目录
cp /export/server/hbase-2.1.0/conf/hbase-site.xml /export/server/apache-phoenix-5.0.0-HBase-2.0-bin/bin/

# 5. 重新启动HBase
stop-hbase.sh
start-hbase.sh

# 6. 启动Phoenix客户端，连接Phoenix Server 第一次启动Phoenix连接HBase会比较慢
cd /export/server/apache-phoenix-5.0.0-HBase-2.0-bin/
bin/sqlline.py zk:2181
```

## 4. phoenix语法

### 4.1 表操作

```bash
# 1. 创建表（订单明细表）
create table if not exists ORDER_DTL(
    id varchar primary key,
    C1.status varchar,
    C1.money double,
    C1.pay_way integer,
    C1.user_id varchar,
    C1.operation_time varchar,
    C1.category varchar
);

# 2. 删除表
drop table if exists ORDER_DTL;

# 3. 创建表（小写）
create table if not exists ORDER_DTL(
    "id" varchar primary key,
    "C1"."status" varchar,
    "C1"."money" double,
    "C1"."pay_way" integer,
    "C1"."user_id" varchar,
    "C1"."operation_time" varchar,
    "C1"."category" varchar
);

select "id" from ORDER_DTL;

# 4. 插入一条数据，双引号表示引用一个表或者字段，单引号表示字符串
upsert into "ORDER_DTL" values('000001', '已提交', 4070, 1, '4944191', '2020-04-25 12:09:16', '手机;');

# 5. 更新数据，将ID为'000001'的订单状态修改为已付款
upsert into "ORDER_DTL"("id", "C1"."status") values('000001', '已付款');

# 6. 指定ID查询数据
select * from "ORDER_DTL" where "id" = '000001';

# 7. 删除指定ID的数据
delete from "ORDER_DTL" where "id" = '000001';

# 8. 查询表中一共有多少条数据
select count(*) from "ORDER_DTL";
# 第一页
select * from "ORDER_DTL" limit 10 offset 0;
# 第二页
select * from "ORDER_DTL" limit 10 offset 10;
# 第三页
select * from "ORDER_DTL" limit 10 offset 20;
```

### 4.2 预分区

```bash
# 1. 使用指定rowkey来进行预分区
drop table if exists ORDER_DTL;
create table if not exists ORDER_DTL(
    "id" varchar primary key,
    C1."status" varchar,
    C1."money" float,
    C1."pay_way" integer,
    C1."user_id" varchar,
    C1."operation_time" varchar,
    C1."category" varchar
) 
CONPRESSION='GZ'
SPLIT ON ('3','5','7');

# 2. 直接指定Region的数量来进行预分区
drop table if exists ORDER_DTL;
create table if not exists ORDER_DTL(
    "id" varchar primary key,
    C1."status" varchar,
    C1."money" float,
    C1."pay_way" integer,
    C1."user_id" varchar,
    C1."operation_time" varchar,
    C1."category" varchar
) 
CONPRESSION='GZ',
SALT_BUCKETS=10;

# 3. 创建二级索引，根据用户ID来查询订单的ID以及对应的支付金额，建立一个覆盖索引，加快查询
create index IDX_USER_ID on ORDER_DTL(C1."user_id") include ("id", C1."money");

# 4. 删除索引
drop index IDX_USER_ID on ORDER_DTL;

# 5. 强制使用索引查询
explain select /*+ INDEX(ORDER_DTL IDX_USER_ID) */ * from ORDER_DTL where "user_id" = '8237476';

# 6. 建立本地索引
# 因为我们要在很多的列上建立索引，所以不太使用使用覆盖索引
create local index IDX_LOCAL_ORDER_DTL_MULTI_IDX on ORDER_DTL("id", C1."status", C1."money", C1."pay_way", C1."user_id") ;
explain select * from ORDER_DTL WHERE C1."status" = '已提交';
explain select * from ORDER_DTL WHERE C1."pay_way" = 1;

```

### 4.3 视图映射

```bash
# 1. 建立HBase已经有的表和Phoenix视图的映射
create view if not exists "MOMO_CHAT"."MSG"(
    id varchar primary key,
    "C1"."msg_time" varchar,
    "C1"."sender_nickyname" varchar,
    "C1"."sender_account" varchar,
    "C1"."sender_sex" varchar,
    "C1"."sender_ip" varchar,
    "C1"."sender_os" varchar,
    "C1"."sender_phone_type" varchar,
    "C1"."sender_network" varchar,
    "C1"."sender_gps" varchar,
    "C1"."receiver_nickyname" varchar,
    "C1"."receiver_ip" varchar,
    "C1"."receiver_account" varchar,
    "C1"."receiver_os" varchar,
    "C1"."receiver_phone_type" varchar,
    "C1"."receiver_network" varchar,
    "C1"."receiver_gps" varchar,
    "C1"."receiver_sex" varchar,
    "C1"."msg_type" varchar,
    "C1"."distance" varchar,
    "C1"."message" varchar
);

# 查询一条数据
select * from "MOMO_CHAT"."MSG" limit 1;

# 根据日期、发送人账号、接收人账号查询历史消息
#日期查询：2020-09-10 11:28:05
select
    *
from
    "MOMO_CHAT"."MSG"
where
    substr("msg_time", 0, 10) = '2020-09-10'
and "sender_account" = '13514684105'
and "receiver_account" = '13869783495';

-- 10 rows selected (5.648 seconds)

select * from "MOMO_CHAT"."MSG" where substr("msg_time", 0, 10) = '2020-09-10' and "sender_account" = '13514684105' and "receiver_account" = '13869783495';

# 2. 创建本地索引
CREATE LOCAL INDEX LOCAL_IDX_MOMO_MSG ON MOMO_CHAT.MSG(substr("msg_time", 0, 10), "sender_account", "receiver_account");
drop index LOCAL_IDX_MOMO_MSG ON MOMO_CHAT.MSG;

0 rows selected (0.251 seconds)

explain select * from "MOMO_CHAT"."MSG" where substr("msg_time", 0, 10) = '2020-09-10' and "sender_account" = '13514684105' and "receiver_account" = '13869783495';

# 3. 删除索引
drop index LOCAL_IDX_MOMO_MSG on MOMO_CHAT.MSG;
```