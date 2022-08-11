# HBase 2.4.11

## 1. 安装

### 1.1 上传安装包

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
# 第28行
export JAVA_HOME=/usr/local/jdk1.8.0_202
export HBASE_MANAGES_ZK=false
```

#### 1.2.2 hbase-site.xml

```
vim hbase-site.xml
```

```xml
<configuration>
    <!-- HBase数据在HDFS中的存放的路径 -->
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://node01.xuzhihao.net:8020/hbase</value>
    </property>
    <!-- Hbase的运行模式。false是单机模式，true是分布式模式。若为false,Hbase和Zookeeper会运行在同一个JVM里面 -->
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <!-- ZooKeeper的地址 -->
    <property>
        <name>hbase.zookeeper.quorum</name>
        <value>node01.xuzhihao.net,node02.xuzhihao.net,node03.xuzhihao.net</value>
    </property>
    <!-- ZooKeeper快照的存储位置 -->
    <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/opt/data/zookeeper/data</value>
    </property>
    <!--  V2.1版本，在分布式情况下, 设置为false -->
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

### 1.3 设置Hbase环境变量

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
```

### 1.4 解决版本冲突

```bash
cp $HBASE_HOME/lib/client-facing-thirdparty/htrace-core-3.1.0-incubating.jar $HBASE_HOME/lib/
```

### 1.5 分发安装包

```bash
cd /opt/hbase-2.4.11
scp -r /opt/hbase-2.4.11 node02.xuzhihao.net:$PWD
scp -r /opt/hbase-2.4.11 node03.xuzhihao.net:$PWD
```

### 1.6 启动服务

```bash
cd /export/onekey
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

## 2. 命令

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

## 3. Phoenix 5.1.2

### 3.1 安装

http://phoenix.apache.org/download.html

#### 3.1.1 上传安装包

```bash
cd /opt/software
tar -xvzf apache-phoenix-5.0.0-HBase-2.0-bin.tar.gz -C /opt
mv /opt/phoenix-hbase-2.4-5.1.2-bin /opt/phoenix-hbase-2.4-5.1.2
```

#### 3.1.2 拷贝jar包

```bash
cp /opt/phoenix-hbase-2.4-5.1.2/phoenix-*.jar /opt/hbase-2.4.11/lib/  # 拷贝jar包到hbase lib目录 
# 分发jar
cd /opt/hbase-2.4.11/lib/
scp phoenix-*.jar node02.xuzhihao.net:$PWD
scp phoenix-*.jar node03.xuzhihao.net:$PWD
```

#### 3.1.3 修改配置文件

```
cd /opt/hbase-2.4.11/conf/
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

将配置后的hbase-site.xml拷贝到phoenix的bin目录

```bash
cp /opt/hbase-2.4.11/conf/hbase-site.xml /opt/phoenix-hbase-2.4-5.1.2/bin/
```

#### 3.1.4 配置分发

```bash
# 将hbase-site.xml分发到每个节点
scp hbase-site.xml node02.xuzhihao.net:$PWD
scp hbase-site.xml node03.xuzhihao.net:$PWD
```

#### 3.1.5 重启HBase

```bash
stop-hbase.sh
start-hbase.sh
```

#### 3.1.6 启动Phoenix客户端

```bash
cd /opt/phoenix-hbase-2.4-5.1.2
bin/sqlline.py zk:2181
```

#### 3.1.7 数据验证



### 3.2 语法

#### 3.2.1 表操作

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

#### 3.2.2 预分区

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

#### 3.2.3 视图映射

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