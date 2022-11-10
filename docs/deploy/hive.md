# Hive 3.1.2

Apache Hive是一款建立在Hadoop之上的开源数据仓库系统，可以将存储在Hadoop文件中的结构化、半结构化数据文件映射为一张数据库表，基于表提供了一种类似SQL的查询模型，称为Hive查询语言（HQL），用于访问和分析存储在Hadoop文件中的大型数据集。

Hive核心是将HQL转换为MapReduce程序，然后将程序提交到Hadoop群集执行。Hive由Facebook实现并开源。Hive不是分布式安装运行的软件，其分布式的特性主要借由Hadoop完成。包括分布式存储、分布式计算。hive 3.1.2对应hadoop3.1.4

## 1. 安装

### 1.1 准备工作

由于Apache Hive是一款基于Hadoop的数据仓库软件，通常部署运行在Linux系统之上。因此不管使用何种方式配置Hive Metastore，必须要先保证服务器的基础环境正常，Hadoop集群健康可用

集群环境需要保证：时间同步、防火墙关闭、主机Host映射、免密登录、JDK安装

#### 1.1.1 开启httpfs服务

1. hadoop修改core-site.xml

```bash
vi /opt/hadoop-3.1.4/etc/hadoop/core-site.xml
# 添加
<property>
    <name>hadoop.proxyuser.root.hosts</name>
    <value>*</value>
</property>
<property>
    <name>hadoop.proxyuser.root.groups</name>
    <value>*</value>
</property>
```

2. 分发重启服务

```bash
cd /opt/hadoop-3.1.4/etc/hadoop
scp core-site.xml node02:/opt/hadoop-3.1.4/etc/hadoop/
scp core-site.xml node03:/opt/hadoop-3.1.4/etc/hadoop/

cd /opt/hadoop-3.1.4/sbin
./stop-all.sh
./start-all.sh

hdfs --daemon start httpfs  # 启动httpfs 端口14000
```

### 1.2 内嵌模式

#### 1.2.1 上传解压

```bash
cd /opt/software
tar zxvf apache-hive-3.1.2-bin.tar.gz -C /opt
cd /opt
mv apache-hive-3.1.2-bin apache-hive-3.1.2
```

#### 1.2.2 解决版本冲突

```bash
cd /opt/apache-hive-3.1.2
rm -rf lib/guava-19.0.jar
cp /opt/hadoop-3.1.4/share/hadoop/common/lib/guava-27.0-jre.jar ./lib/
```

#### 1.2.3 修改配置

```bash
cd /opt/apache-hive-3.1.2/conf/
mv hive-env.sh.template hive-env.sh
vim hive-env.sh
# 编辑内容
export HADOOP_HOME=/opt/hadoop-3.1.4
export HIVE_CONF_DIR=/opt/apache-hive-3.1.2/conf
export HIVE_AUX_JARS_PATH=/opt/apache-hive-3.1.2/lib
```

#### 1.2.4 设置日志保存路径

```bash
cd /opt/apache-hive-3.1.2/conf/
mv hive-log4j2.properties.template hive-log4j2.properties
vim hive-log4j2.properties
# 编辑
property.hive.log.dir = /opt/apache-hive-3.1.2/logs
```

#### 1.2.5 初始化metadata

```bash
cd /opt/apache-hive-3.1.2
bin/schematool -dbType derby -initSchema
```

#### 1.2.6 启动hive服务

```bash
cd /opt/apache-hive-3.1.2
bin/hive
```

#### 1.2.7 客户端测试

```sql
show databases;
create database test;
show tables;
```

### 1.3 本地模式

#### 1.3.1 安装MySQL

[MySQL 5.7.29](devops/database/mysql?id=_11-单机)

#### 1.3.2 上传解压

```bash
cd /opt/software
tar zxvf apache-hive-3.1.2-bin.tar.gz -C /opt
cd /opt
mv apache-hive-3.1.2-bin apache-hive-3.1.2
```

#### 1.3.3 解决版本冲突

```bash
cd /opt/apache-hive-3.1.2
rm -rf lib/guava-19.0.jar
cp /opt/hadoop-3.1.4/share/hadoop/common/lib/guava-27.0-jre.jar ./lib/

# mysql驱动拷贝到hive安装包lib/文件下
cp /opt/software/mysql/mysql-connector-java-5.1.32.jar ./lib/
```

#### 1.3.4 修改配置

1. 设置hive环境变量

```bash
cd /opt/apache-hive-3.1.2/conf/
mv hive-env.sh.template hive-env.sh
vim hive-env.sh
# 编辑内容
export HADOOP_HOME=/opt/hadoop-3.1.4
export HIVE_CONF_DIR=/opt/apache-hive-3.1.2/conf
export HIVE_AUX_JARS_PATH=/opt/apache-hive-3.1.2/lib
```

2. 新增hive-site.xml配置mysql

```bash
cd /opt/apache-hive-3.1.2/conf/
vim hive-site.xml

# 插入
<configuration>
    <!-- 存储元数据mysql相关配置 -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://node01:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hadoop</value>
    </property>

    <!-- 关闭元数据存储授权  -->
    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>

    <!-- 关闭元数据存储版本的验证 -->
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
</configuration>
```


#### 1.3.5 初始化metadata

```bash
cd /opt/apache-hive-3.1.2
bin/schematool -initSchema -dbType mysql -verbos
```

初始化成功会在mysql中创建74张表

#### 1.3.6 启动hive服务

```bash
cd /opt/apache-hive-3.1.2
bin/hive
```

#### 1.3.7 客户端测试

```sql
show databases;
create database test;
show tables;
```

### 1.4 远程模式

#### 1.4.1 上传解压

```bash
cd /opt/software
tar zxvf apache-hive-3.1.2-bin.tar.gz -C /opt
cd /opt
mv apache-hive-3.1.2-bin apache-hive-3.1.2
```

#### 1.4.2 解决版本冲突

```bash
cd /opt/apache-hive-3.1.2
rm -rf lib/guava-19.0.jar
cp /opt/hadoop-3.1.4/share/hadoop/common/lib/guava-27.0-jre.jar ./lib/

# mysql驱动拷贝到hive安装包lib/文件下
cp /opt/software/mysql/mysql-connector-java-5.1.32.jar ./lib/
```

#### 1.4.3 修改配置

1. 设置hive环境变量

```bash
cd /opt/apache-hive-3.1.2/conf/
mv hive-env.sh.template hive-env.sh
vim hive-env.sh
# 编辑内容
export HADOOP_HOME=/opt/hadoop-3.1.4
export HIVE_CONF_DIR=/opt/apache-hive-3.1.2/conf
export HIVE_AUX_JARS_PATH=/opt/apache-hive-3.1.2/lib
```

2. 新增hive-site.xml配置mysql

```bash
cd /opt/apache-hive-3.1.2/conf/
vim hive-site.xml

# 插入
<configuration>
    <!-- 存储元数据mysql相关配置 -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value> jdbc:mysql://node01:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hadoop</value>
    </property>

    <!-- H2S运行绑定host -->
    <property>
        <name>hive.server2.thrift.bind.host</name>
        <value>node01</value>
    </property>

    <!-- 远程模式部署metastore 服务地址 -->
    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://node01:9083</value>
    </property>

    <!-- 关闭元数据存储授权  -->
    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>

    <!-- 关闭元数据存储版本的验证 -->
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
</configuration>
```


#### 1.4.4 初始化metadata

```bash
cd /opt/apache-hive-3.1.2
bin/schematool -initSchema -dbType mysql -verbos
```

初始化成功会在mysql中创建74张表

#### 1.4.5 启动服务

1. 启动metastore

```bash
/opt/apache-hive-3.1.2/bin/hive --service metastore             # 前台启动
/opt/apache-hive-3.1.2/bin/hive --service metastore --hiveconf hive.root.logger=DEBUG,console   # 前台启动开启debug日志
nohup /opt/apache-hive-3.1.2/bin/hive --service metastore &     # 后台启动
```

2. 启动hiveserver2

```bash
nohup /opt/apache-hive-3.1.2/bin/hive --service hiveserver2  & 
```

## 2. 客户端

### 2.1 beeline

#### 2.1.1 上传解压

```bash
cd /opt/software
tar zxvf apache-hive-3.1.2-bin.tar.gz -C /opt
cd /opt
mv apache-hive-3.1.2-bin apache-hive-3.1.2
```

#### 2.2.2 解决版本冲突

```bash
cd /opt/apache-hive-3.1.2
rm -rf lib/guava-19.0.jar
cp /opt/hadoop-3.1.4/share/hadoop/common/lib/guava-27.0-jre.jar ./lib/
```

#### 2.2.3 修改配置

1. 设置hive环境变量

```bash
cd /opt/apache-hive-3.1.2/conf/
mv hive-env.sh.template hive-env.sh
vim hive-env.sh
# 编辑内容
export HADOOP_HOME=/opt/hadoop-3.1.4
export HIVE_CONF_DIR=/opt/apache-hive-3.1.2/conf
export HIVE_AUX_JARS_PATH=/opt/apache-hive-3.1.2/lib
```

2. 新增hive-site.xml配置mysql

```bash
cd /opt/apache-hive-3.1.2/conf/
vim hive-site.xml

# 插入
<configuration>
    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://node01:9083</value>
    </property>
</configuration>
```

#### 2.2.4 测试

```bash
/opt/apache-hive-3.1.2/bin/beeline
! connect jdbc:hive2://node01:10000 # 户名root 密码为空
```

```sql
show databases;
create database test;
use test;
show tables;
create table t_student(id int,name varchar(255));
insert into table t_student values(1,"zhangsan");
select * from t_student;
```

MR验证：http://192.168.2.201:8088/cluster

### 2.2 IDEA

添加插件

![](../../assets/_images/devops/deploy/hive/idea1.png)

设置连接

![](../../assets/_images/devops/deploy/hive/idea2.png)

设置驱动

![](../../assets/_images/devops/deploy/hive/idea3.png)

验证数据

![](../../assets/_images/devops/deploy/hive/idea4.png)

## 3. DDL

完整语法树

```sql
CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name
[(col_name data_type [COMMENT col_comment], ... ]
[COMMENT table_comment]
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
[CLUSTERED BY (col_name, col_name, ...) [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]
[ROW FORMAT DELIMITED|SERDE serde_name WITH SERDEPROPERTIES (property_name=property_value,...)]
[STORED AS file_format]
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)];
```

### 3.1 原生数据类型表

#### 3.1.1 创建表结构

```sql
create table t_user(id int,name varchar(255),age int,city varchar(255))
row format delimited
fields terminated by ',';
```

#### 3.1.2 上传文件

```txt
1,zhangsan,18,beijing
2,lisi,25,shanghai
3,allen,30,shanghai
4,woon,15,nanjing
5,james,45,hangzhou
6,tony,26,beijing
```

```bash
cd /hivedata
vi user.txt

hadoop fs -put user.txt /user/hive/warehouse/test.db/t_user
hadoop fs -ls /user/hive/warehouse/test.db/t_user
```

#### 3.1.3 数据验证

```bash
select * from t_user;
```

### 3.2 复杂数据类型表

#### 3.2.1 创建表结构

```sql
create database honor_of_kings;
use honor_of_kings;

create table t_hot_hero_skin_price(
    id int,
    name string,
    win_rate int,
    skin_price map<string,int>
)
row format delimited
fields terminated by ','            --字段之间分隔符
collection items terminated by '-'  --集合元素之间分隔符
map keys terminated by ':' ;        --集合元素kv之间分隔符;
```

#### 3.2.2 上传文件

```txt
1,孙悟空,53,西部大镖客:288-大圣娶亲:888-全息碎片:0-至尊宝:888-地狱火:1688
2,鲁班七号,54,木偶奇遇记:288-福禄兄弟:288-黑桃队长:60-电玩小子:2288-星空梦想:0
3,后裔,53,精灵王:288-阿尔法小队:588-辉光之辰:888-黄金射手座:1688-如梦令:1314
4,铠,52,龙域领主:288-曙光守护者:1776
5,韩信,52,飞衡:1788-逐梦之影:888-白龙吟:1188-教廷特使:0-街头霸王:888
```

字段：id、name(英雄名称)、win_rate(胜率)、skin_price(皮肤及价格)

```bash
cd /hivedata
vi hot_hero_skin_price.txt

hadoop fs -put hot_hero_skin_price.txt /user/hive/warehouse/honor_of_kings.db/t_hot_hero_skin_price
```

#### 3.2.3 数据验证

```sql
select * from t_hot_hero_skin_price
```

### 3.3 内、外部表

#### 3.3.1 内部表

内部表（Internal table）也称为被Hive拥有和管理的托管表（Managed table）。默认情况下创建的表就是内部表，Hive拥有该表的结构和文件，类似于RDBMS中的表

#### 3.3.2 外部表

外部表（External table）中的数据不是Hive拥有或管理的。要创建一个外部表，需要使用EXTERNAL语法关键字。删除外部表只会删除元数据，而不会删除实际数据。在Hive外部仍然可以访问实际数据。`External`关键字与是否使用`location`关键字无关。

1. 创建表

```sql
create external table student_ext(
    num int,
    name string,
    sex string,
    age int,
    dept string)
row format delimited
fields terminated by ','
location '/stu';
```

2. 查看表信息

```sql
describe formatted student_ext
```

### 3.4 分区表

#### 3.4.1 单分区表

1. 以role角色作为分区字段创建表

```sql
create table t_all_hero_part(
       id int,
       name string,
       hp_max int,
       mp_max int,
       attack_max int,
       defense_max int,
       attack_range string,
       role_main string,
       role_assist string
) partitioned by (role string)
row format delimited
fields terminated by "\t";
```

2. 上传数据

战士warrior.txt

```txt
52	刘备	6900	1742	363	381	remotely	warrior
53	钟无艳	7000	1760	318	409	melee	warrior	tank
54	吕布	7344	0	343	390	melee	warrior	tank
55	亚瑟	8050	0	346	400	melee	warrior	tank
56	雅典娜	6264	1732	327	418	melee	warrior	tank
57	关羽	7107	10	343	386	melee	warrior	tank
58	露娜	6612	1836	335	375	melee	warrior	mage
59	花木兰	5397	100	362	349	melee	warrior	assassin
60	赵云	6732	1760	380	394	melee	warrior	assassin
61	杨戬	7420	1694	325	428	melee	warrior
62	达摩	7140	1694	355	415	melee	warrior
63	孙悟空	6585	1760	349	385	melee	warrior	assassin
64	曹操	7473	0	361	371	melee	warrior
65	典韦	7516	1774	345	402	melee	warrior
66	宫本武藏	6210	0	330	391	melee	warrior
67	老夫子	7155	5	329	409	melee	warrior
68	哪吒	7268	1808	320	408	melee	warrior
69	铠	6700	1784	328	388	melee	warrior	tank
```

坦克tank.txt

```txt
42	夏侯惇	7350	1746	321	397	melee	tank	warrior
43	张飞	8341	100	301	504	melee	tank	support
44	牛魔	8476	1926	273	394	melee	tank	support
45	程咬金	8611	0	316	504	melee	tank	warrior
46	廉颇	9328	1708	286	514	melee	tank
47	东皇太一	7669	1926	286	360	melee	tank
48	白起	8638	1666	288	430	melee	tank
49	刘邦	8073	1940	302	504	melee	tank	support
50	刘禅	8581	1694	295	459	melee	tank
51	项羽	8057	1694	306	494	melee	tank
```

法师mage.txt

```txt
17	芈月	6164	100	289	361	remotely	mage	tank
18	嬴政	5471	1946	309	295	remotely	mage
19	武则天	5037	1988	297	348	remotely	mage
20	甄姬	5584	2002	296	330	remotely	mage
21	妲己	5824	2016	293	326	remotely	mage
22	干将莫邪	5583	1946	292	323	remotely	mage
23	姜子牙	5399	2002	317	342	remotely	mage	support
24	王昭君	5429	1960	296	305	remotely	mage
25	诸葛亮	5655	1988	287	330	remotely	mage
26	安琪拉	5994	1960	293	315	remotely	mage
27	小乔	5916	1988	263	309	remotely	mage
28	周瑜	5513	1974	298	320	remotely	mage
29	张良	5799	1988	293	320	remotely	mage
30	高渐离	6165	1988	290	343	remotely	mage
31	扁鹊	6703	2016	309	374	remotely	mage	support
32	墨子	7176	1722	328	475	melee	mage	tank
33	不知火舞	6014	200	293	336	melee	mage	assassin
34	貂蝉	5611	1960	287	330	melee	mage	assassin
35	钟馗	6280	1988	278	390	melee	mage	warrior
```

3. 静态分区

```bash
load data local inpath '/hivedata/warrior.txt' into table t_all_hero_part partition(role='zhanshi');
load data local inpath '/hivedata/mage.txt' into table t_all_hero_part partition(role='fashi');
load data local inpath '/hivedata/tank.txt' into table t_all_hero_part partition(role='tanke');
```

4. 数据验证

```sql
--先基于分区过滤 再查询
select count(*) from t_all_hero_part where role="zhanshi" and hp_max >6000;
```

#### 3.4.2 双分区表

1. 创建分区表

```sql
--双分区表，按省份和市分区
create table t_user_province_city (id int, name string,age int) partitioned by (province string, city string);
--双分区表的使用  使用分区进行过滤 减少全表扫描 提高查询效率
select * from t_user_province_city where  province= "zhejiang" and city ="hangzhou";
```

#### 3.4.3 动态分区表

1. 设置开启动态分区

```bash
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
```

2. 创建动态分区表

```sql
create table t_all_hero_part_dynamic(
    id int,
    name string,
    hp_max int,
    mp_max int,
    attack_max int,
    defense_max int,
    attack_range string,
    role_main string,
    role_assist string
) partitioned by (role string)
row format delimited
fields terminated by "\t";
```

3. 提前将所有数据导入到临时表中

```sql
create table t_all_hero(
   id int,
   name string,
   hp_max int,
   mp_max int,
   attack_max int,
   defense_max int,
   attack_range string,
   role_main string,
   role_assist string
)
row format delimited
fields terminated by "\t";
```

```bash
hadoop fs -put warrior.txt warrior.txt mage.txt /user/hive/warehouse/test.db/t_all_hero
```

4. 执行动态分区插入

```sql
insert into table t_all_hero_part_dynamic partition(role) 
select tmp.*,tmp.role_main from t_all_hero tmp;
```

5. 数据验证

```sql
select * from t_all_hero_part_dynamic
```

### 3.5 分桶表

#### 3.5.1 开启分桶

```bash
#从Hive2.0开始不再需要设置
set hive.enforce.bucketing=true;
```

#### 3.5.2 创建分桶表

```sql
CREATE TABLE test.t_usa_covid19_bucket(
      count_date string,
      county string,
      state string,
      fips int,
      cases int,
      deaths int)
CLUSTERED BY(state) INTO 5 BUCKETS; --分桶的字段一定要是表中已经存在的字段


--根据state州分为5桶 每个桶内根据cases确诊病例数倒序排序
CREATE TABLE test.t_usa_covid19_bucket_sort(
     count_date string,
     county string,
     state string,
     fips int,
     cases int,
     deaths int)
CLUSTERED BY(state)
sorted by (cases desc) INTO 5 BUCKETS;--指定每个分桶内部根据 cases倒序排序
```

#### 3.5.3 上传数据

数据样例：

```txt
2021-01-28,La Plata,Colorado,08067,2673,34
2021-01-28,Lake,Colorado,08065,502,0
2021-01-28,Larimer,Colorado,08069,17834,189
2021-01-28,Las Animas,Colorado,08071,907,11
2021-01-28,Lincoln,Colorado,08073,889,3
2021-01-28,Logan,Colorado,08075,3610,58
2021-01-28,Mesa,Colorado,08077,11822,168
2021-01-28,Mineral,Colorado,08079,57,1
2021-01-28,Moffat,Colorado,08081,617,22
2021-01-28,Montezuma,Colorado,08083,1616,21
2021-01-28,Montrose,Colorado,08085,2956,43
2021-01-28,Morgan,Colorado,08087,2349,88
2021-01-28,Otero,Colorado,08089,1786,59
2021-01-28,Ouray,Colorado,08091,195,3
2021-01-28,Park,Colorado,08093,479,5
```

上传文件

```bash
hadoop fs -put us-covid19-counties.dat /user/hive/warehouse/test.db/t_usa_covid19
```

把源数据加载到普通hive表中

```sql
drop table if exists t_usa_covid19;
CREATE TABLE itheima.t_usa_covid19(
       count_date string,
       county string,
       state string,
       fips int,
       cases int,
       deaths int)
row format delimited fields terminated by ",";
```

#### 3.5.4 数据转换

```sql
insert into t_usa_covid19_bucket select * from t_usa_covid19;
```

#### 3.5.5 数据验证

除了数据的验证，还需要到hadoop文件夹下验证是否按照分桶表结构将文件拆分成小文件

```sql
select * from t_usa_covid19_bucket;
select * from t_usa_covid19_bucket where state="New York";
```

### 3.6 事务表

#### 3.6.1 开启事务

```bash
# 可以使用set设置当前session生效 也可以配置在hive-site.xml中
set hive.support.concurrency = true;        # Hive是否支持并发
set hive.enforce.bucketing = true;          # 从Hive2.0开始不再需要，是否开启分桶功能
set hive.exec.dynamic.partition.mode = nonstrict;                       # 动态分区模式  非严格
set hive.txn.manager = org.apache.hadoop.hive.ql.lockmgr.DbTxnManager;  # 事务处理类
set hive.compactor.initiator.on = true;                                 # 是否在Metastore实例上运行启动线程和清理线程
set hive.compactor.worker.threads = 1;                                  # 在此metastore实例上运行多少个压缩程序工作线程。
```

#### 3.6.2 创建事务表

```sql
drop table if exists trans_student;
create table trans_student(
   id int,
   name String,
   age int
)clustered by (id) into 2 buckets stored as orc TBLPROPERTIES('transactional'='true');
```

注意：事务表创建几个要素：开启参数、分桶表、存储格式orc、表属性


### 3.7 视图

#### 3.7.1 创建视图

```sql
create view v_usa_covid19 as select count_date, county,state,deaths from t_usa_covid19 limit 5;
show create table v_usa_covid19;    --查看视图定义
show views; --hive v2.2.0之后支持
```

#### 3.7.2 修改视图

```sql
drop view v_usa_covid19_from_view;                                              --删除视图
alter view v_usa_covid19 set TBLPROPERTIES ('comment' = 'This is a view');      --更改视图属性
alter view v_usa_covid19 as  select county,deaths from t_usa_covid19 limit 2;   --更改视图定义
```

### 3.8 物化视图

#### 3.8.1 开启事务

```bash
# 可以使用set设置当前session生效 也可以配置在hive-site.xml中
set hive.support.concurrency = true;        # Hive是否支持并发
set hive.enforce.bucketing = true;          # 从Hive2.0开始不再需要，是否开启分桶功能
set hive.exec.dynamic.partition.mode = nonstrict;                       # 动态分区模式  非严格
set hive.txn.manager = org.apache.hadoop.hive.ql.lockmgr.DbTxnManager;  # 事务处理类
set hive.compactor.initiator.on = true;                                 # 是否在Metastore实例上运行启动线程和清理线程
set hive.compactor.worker.threads = 1;                                  # 在此metastore实例上运行多少个压缩程序工作线程。
```

#### 3.8.2 创建事务表

```sql
drop table if exists  student_trans;
CREATE TABLE student_trans (
      sno int,
      sname string,
      sdept string)
clustered by (sno) into 2 buckets stored as orc TBLPROPERTIES('transactional'='true');
```

#### 3.8.3 数据导入

导入数据到student_trans中

```sql
insert overwrite table student_trans
select num,name,dept
from student;

select * from student_trans;
```

#### 3.8.4 创建物化视图

```sql
CREATE MATERIALIZED VIEW student_trans_agg
AS SELECT sdept, count(*) as sdept_cnt from student_trans group by sdept;

show tables;
show materialized views;
```

#### 3.8.5 数据验证

没有创建物化视图之前是MR程序进行聚合，创建以后由于会命中物化视图，重写query查询物化视图，查询速度会加快（没有启动MR，只是普通的table scan）

```sql
SELECT sdept, count(*) as sdept_cnt from student_trans group by sdept;
```

查询执行计划可以发现 查询被自动重写为TableScan alias: test.student_trans_agg，转换成了对物化视图的查询，提高了查询效率

```sql
explain SELECT sdept, count(*) as sdept_cnt from student_trans group by sdept;
```

验证禁用物化视图自动重写

```sql
ALTER MATERIALIZED VIEW student_trans_agg DISABLE REWRITE;
```

删除物化视图

```sql
drop materialized view student_trans_agg;
```