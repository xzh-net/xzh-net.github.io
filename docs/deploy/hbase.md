# HBase 2.4.11

Apache Hbase是一个开源的、分布式的、带版本的、非关系型数据库，模仿谷歌的BigTable。BigTable使用Google File System作为分布式数据存储，同理Hbase使用HDFS，当你需要随机地实时读写大数据时使用Hbase。它的目标是管理超级大表-数十亿行X数百万列

## 1. 安装

### 1.1 上传解压

```bash
mkdir -p /opt/software
cd /opt/software 
tar -xvzf hbase-2.4.11-bin.tar.gz -C /opt
```

### 1.2 修改配置

#### 1.2.1 hbase-env.sh

```bash
cd /opt/hbase-2.4.11/conf
vim hbase-env.sh
# 编辑内容
export JAVA_HOME=/usr/local/jdk1.8.0_202
export HBASE_MANAGES_ZK=false
```

#### 1.2.2 hbase-site.xml

```
vim hbase-site.xml
```

```xml
<configuration>
    <!-- Hbase的运行模式。false是单机模式，true是分布式模式。若为false,Hbase和Zookeeper会运行在同一个JVM里面 -->
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <!-- HBase数据在HDFS中的存放的路径 -->
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://node01.xuzhihao.net:8020/hbase</value>
    </property>
    <!-- ZooKeeper的地址 -->
    <property>
        <name>hbase.zookeeper.quorum</name>
        <value>node01.xuzhihao.net,node02.xuzhihao.net,node03.xuzhihao.net</value>
    </property>
    <!--Hbase临时数据存储的地方 默认是 ./tmp -->
    <property>
        <name>hbase.tmp.dir</name>
        <value>/opt/hbase-2.4.11/data</value>
    </property>
    <!--  分布式情况下, 设置为false -->
    <property>
        <name>hbase.unsafe.stream.capability.enforce</name>
        <value>false</value>
    </property>
</configuration>
```

#### 1.2.3 修改regionservers

```bash
cd /opt/hbase-2.4.11/conf
vim regionservers
```

```bash
node01.xuzhihao.net
node02.xuzhihao.net
node03.xuzhihao.net
```

### 1.3 设置环境变量

```bash
vim /etc/profile
export HBASE_HOME=/opt/hbase-2.4.11
export PATH=$PATH:${HBASE_HOME}/bin:${HBASE_HOME}/sbin
# 加载环境变量
source /etc/profile

scp /etc/profile root@node02:/etc/
scp /etc/profile root@node03:/etc/

# 三台机器验证
source /etc/profile
hbase
```

### 1.4 解决版本冲突

解决HBase和Hadoop的log4j兼容性问题，修改HBase的jar包，使用Hadoop的jar

```bash
mv /opt/hbase-2.4.11/lib/client-facing-thirdparty/slf4j-reload4j-1.7.33.jar /opt/hbase-2.4.11/lib/client-facing-thirdparty/slf4j-reload4j-1.7.33.jar.bak
```

### 1.5 分发安装包

```bash
cd /opt/hbase-2.4.11
scp -r /opt/hbase-2.4.11 node02.xuzhihao.net:$PWD
scp -r /opt/hbase-2.4.11 node03.xuzhihao.net:$PWD
```

### 1.6 启动服务

1. 单点启动

```bash
hbase-daemon.sh start master
hbase-daemon.sh start regionserver
```

2. 集群启动

```bash
/opt/shell/zk.sh start  # 启动zk
start-dfs.sh            # 启动hadoop
start-hbase.sh          # 启动hbase
```

### 1.7 客户端测试

```bash
# 启动hbase shell客户端
hbase shell
# 输入status
```

WebUI：http://node01.xuzhihao.net:16010/master-status


### 1.8 Master高可用

#### 1.8.1 停止服务

```bash
stop-hbase.sh
```

#### 1.8.2 添加Master节点

```bash
cd /opt/hbase-2.4.11/conf
vi backup-masters
# 添加
node02.xuzhihao.net
node03.xuzhihao.net
```

#### 1.8.3 配置分发

```bash
scp backup-masters node02:/opt/hbase-2.4.11/conf
scp backup-masters node03:/opt/hbase-2.4.11/conf/
```

#### 1.8.4 验证高可用

```bash
start-hbase.sh
```

启动集群后，在node02出现HMaster节点，访问地址：http://node02.xuzhihao.net:16010/master-status， 不过此时处于备用状态，当手动kill掉node01机器的HMaster节点，node02的HMaster节点变成可用状态

```bash
hbase-daemon.sh start master    # 重启node01上的HMaster
```


## 2. 命令

### 2.1 DDL

#### 2.1.1 命名空间

```shell
create_namespace 'xzh'         # 创建命名空间
list_namespace                 # 查看所有命名空间
drop_namespace 'xzh'           # 删除命名空间
describe_namespace 'xzh'       # 查看某个具体的命名空间
```

#### 2.1.2 表操作

```shell
create "ORDER_INFO", 'B1', 'C1'         # 创建订单表ORDER_INFO，有一个列蔟为B1，C1
create "xzh:MSG", "C1"                  # 指定命名空间创建表
describe 'xzh:MSG'                      # 查看表详细信息
alter 'ORDER_INFO', 'C2'                # 新增列蔟C3
alter 'ORDER_INFO', 'delete' => 'C3'    # 删除列簇 C3
alter "xzh:MSG", {NAME => "C1", COMPRESSION => "GZ"}  # 指定修改某个表的列蔟，它的压缩方式
create 'xzh:MSG', {NAME => "C1", COMPRESSION => "GZ"}, { NUMREGIONS => 6, SPLITALGO => 'HexStringSplit'} # 在创建表时需要指定预分区
disable "ORDER_INFO"                    # 禁用表
drop "ORDER_INFO"                       # 删除表
```

### 2.2 DML

#### 2.2.1 插入数据

```shell
# 添加数据 put '表名','ROWKEY','列蔟名:列名','值'
put "ORDER_INFO", "000001", "C1:STATUS", "已提交"
put "ORDER_INFO", "000001", "C1:PAYWAY", "1"
put "ORDER_INFO", "000001", "C1:PAY_MONEY", 4070
put "ORDER_INFO", "000001", "C1:USER_ID", "4944191"
put "ORDER_INFO", "000001", "C1:OPERATION_DATE", "2020-04-25 12:09:16"
```

#### 2.2.2 读取数据

```shell
# 单条查询
get "ORDER_INFO", "000001" # 查询ORDER_INFO rowkey为：000001
get "ORDER_INFO", "000001", {FORMATTER => 'toString'}   # 指定字符集
get "ORDER_INFO", "000001", {COLUMN => 'C1:PAY_MONEY'}  # 查询指定列族下指定列的值
count "ORDER_INFO"  # 查询记录数
```

```shell
# scan操作
scan "ORDER_INFO", {FORMATTER => 'toString'}                # 查询订单所有数据
scan "ORDER_INFO", {FORMATTER => 'toString', LIMIT => 3}    # 查询订单数据（只显示3条）
scan "ORDER_INFO", {FORMATTER => 'toString', LIMIT => 3, COLUMNS => ['C1:STATUS', 'C1:OPERATION_DATE']}                         # 显示指定列，只显示3条
scan "ORDER_INFO", {ROWPREFIXFILTER => '000001',FORMATTER => 'toString', LIMIT => 3, COLUMNS => ['C1:STATUS', 'C1:PAYWAY']}     # {ROWPREFIXFILTER => 'rowkey'}
scan "ORDER_INFO", {FILTER => "RowFilter(=,'binary:000001')", COLUMNS => ['C1:STATUS', 'C1:PAYWAY'], FORMATTER => 'toString'}   # Scan + Filter 只查询订单的ID为：000001、订单状态以及支付方式
scan "ORDER_INFO", {FILTER => "SingleColumnValueFilter('C1', 'STATUS', =, 'binary:已付款')", FORMATTER => 'toString'}            # 查询状态为「已付款」的订单
scan "ORDER_INFO", {FILTER => "SingleColumnValueFilter('C1', 'PAYWAY', =, 'binary:1') AND SingleColumnValueFilter('C1', 'PAY_MONEY', >, 'binary:3000')", FORMATTER => 'toString'}   # 查询支付方式为1，且金额大于3000的订单
```

#### 2.2.3 删除数据

```shell
# 删除操作
delete "ORDER_INFO", "000001", "C1:STATUS" # 删除列
deleteall "ORDER_INFO", "000001"    # 删除所有列
```

## 3. Phoenix 5.1.2

### 3.1 安装

http://phoenix.apache.org/download.html

#### 3.1.1 上传安装包

```bash
cd /opt/software
tar -xvzf phoenix-hbase-2.4-5.1.2-bin.tar.gz -C /opt
mv /opt/phoenix-hbase-2.4-5.1.2-bin /opt/phoenix-hbase-2.4-5.1.2
```

#### 3.1.2 拷贝jar包

```bash
# 拷贝jar包到hbase，lib目录 
cp /opt/phoenix-hbase-2.4-5.1.2/phoenix-*.jar /opt/hbase-2.4.11/lib/  

# 分发lib
cd /opt/hbase-2.4.11/lib/
scp phoenix-*.jar node02.xuzhihao.net:$PWD
scp phoenix-*.jar node03.xuzhihao.net:$PWD
```

#### 3.1.3 设置环境变量

```bash
vi /etc/profile
export PHOENIX_HOME=/opt/phoenix-hbase-2.4-5.1.2
export PHOENIX_CLASSPATH=$PHOENIX_HOME
export PATH=$PATH:$PHOENIX_HOME/bin
source /etc/profile

scp /etc/profile root@node02:/etc/
scp /etc/profile root@node03:/etc/
```

#### 3.1.4 二级索引

```
cd /opt/hbase-2.4.11/conf/
vim hbase-site.xml
```

```xml
<!-- 支持索引预写日志编码 -->
<property>
    <name>hbase.regionserver.wal.codec</name>
    <value>org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec</value>
</property>
```

将配置后的hbase-site.xml拷贝到phoenix的bin目录

```bash
cp /opt/hbase-2.4.11/conf/hbase-site.xml /opt/phoenix-hbase-2.4-5.1.2/bin/
```

#### 3.1.5 配置分发

1. 分发hbase

```bash
cd /opt/hbase-2.4.11/conf
scp -r hbase-site.xml node02.xuzhihao.net:$PWD
scp -r hbase-site.xml node03.xuzhihao.net:$PWD
```

2. 分发phoenix

```bash
cd /opt/phoenix-hbase-2.4-5.1.2
scp -r /opt/phoenix-hbase-2.4-5.1.2 node02.xuzhihao.net:$PWD
scp -r /opt/phoenix-hbase-2.4-5.1.2 node03.xuzhihao.net:$PWD
```

#### 3.1.6 重启HBase

```bash
stop-hbase.sh
start-hbase.sh
```

#### 3.1.7 连接Phoenix

```bash
cd /opt/phoenix-hbase-2.4-5.1.2
bin/sqlline.py node01:2181
```

### 3.2 命令

https://phoenix.apache.org/language/index.html

#### 3.2.1 表操作

```sql
-- 创建表，直接指定单个列作为 RowKey，在phoenix中，表名等会自动转换为大写，若要小写，使用双引号
CREATE TABLE IF NOT EXISTS student(
id VARCHAR primary key,
name VARCHAR,
age BIGINT,
addr VARCHAR);

drop table if exists student; -- 删除表

-- 创建表，指定多个列的联合作为 RowKey
CREATE TABLE IF NOT EXISTS "student" (
id VARCHAR NOT NULL,
name VARCHAR NOT NULL,
age BIGINT,
addr VARCHAR
CONSTRAINT my_pk PRIMARY KEY (id, name)) column_encoded_bytes=0;

-- 插入数据
upsert into student values('1001','zhangsan', 10, 'beijing');
-- 查询记录
select * from student;
select * from student where id='1001';
select count(*) from student;
select * from student limit 10 offset 0;
-- 删除记录
delete from student where id='1001';
-- 删除表
drop table student;
```

#### 3.2.2 预分区

1. 使用指定rowkey来进行预分区

```sql
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
```

2. 指定Region的数量来进行预分区

```sql
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
```

#### 3.2.3 视图映射

1. 只读视图

Phoenix创建的视图是只读的，所以只能用来做查询，无法通过视图对数据进行修改等操作

```sql
create view if not exists "test" (
    id varchar primary key,
    "info1"."name" varchar,
    "info2"."address" varchar
);
```

2. 映射已有表

创建Phoenix表来隐射HBase中已经存在的数据。删除Phoenix中的表，HBase中被映射的表也会被删除

```sql
create 'test','info1','info2'  -- hbase中的表
```

```sql
create table "test" (
    id varchar primary key,
    "info1"."cdate" varchar,
    "info1"."name" varchar,
    "info2"."address" varchar
) column_encoded_bytes=0;

CREATE LOCAL INDEX LOCAL_IDX_TEST ON test(substr("cdate", 0, 10), "name");  -- 创建本地索引
drop index LOCAL_IDX_TEST on test;  -- 删除索引
explain select * from "test" where substr("cdate", 0, 10) = '2022-08-15' and "name" = 'zhangsan' ; -- 执行计划
```