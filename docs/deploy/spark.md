# Spark 3.1.3

Apache Spark是一个快速的，用于海量数据处理的通用引擎

## 1. 安装

### 1.1 Local模式

#### 1.1.1 上传解压

下载地址：https://spark.apache.org/downloads.html

```bash
mkdir -p /opt/software
cd /opt/software
tar -zxvf spark-3.1.3-bin-hadoop3.2.tgz -C /opt
mv /opt/spark-3.1.3-bin-hadoop3.2 /opt/spark
```

#### 1.1.2 修改权限

```bash
chown -R root /opt/spark
chgrp -R root /opt/spark
```

#### 1.1.3 客户端测试

1. 启动spark交互式窗口

```bash
/opt/spark/bin/spark-shell
```

2. 测试文件

```bash
vim /opt/words.txt
```

```
hello me you her
hello me you
hello me
hello
```

3. 执行

```bash
val textFile = sc.textFile("file:///opt/words.txt")
val counts = textFile.flatMap(_.split(" ")).map((_,1)).reduceByKey(_ + _)
counts.collect
```

### 1.2 StandAlone模式

#### 1.2.1 集群规划

```lua
node01  master
node02  worker/slave
node03  worker/slave
```

#### 1.2.2 配置workers

```bash
cd /opt/spark/conf
cp workers.template workers
vim workers
# 添加内容
node02
node03
```

#### 1.2.3 配置master

```bash
cd /opt/spark/conf
cp spark-env.sh.template spark-env.sh
vim spark-env.sh
```

```sh
## 设置JAVA安装目录
JAVA_HOME=/usr/local/jdk1.8.0_202

## HADOOP软件配置文件目录，读取HDFS上文件和运行Spark在YARN集群时需要,先提前配上
HADOOP_CONF_DIR=/opt/hadoop-3.1.4/etc/hadoop
YARN_CONF_DIR=/opt/hadoop-3.1.4/etc/hadoop

## 指定spark老大Master的IP和提交任务的通信端口
SPARK_MASTER_HOST=node01
SPARK_MASTER_PORT=7077

SPARK_MASTER_WEBUI_PORT=8080

SPARK_WORKER_CORES=1
SPARK_WORKER_MEMORY=1g
```

#### 1.2.4 分发安装包

```bash
cd /opt/spark/
scp -r /opt/spark root@node02:$PWD
scp -r /opt/spark root@node03:$PWD
```

#### 1.2.5 启动集群

1. 启停命令

```bash
/opt/spark/sbin/start-all.sh  # 在主节点上启动spark集群
/opt/spark/sbin/stop-all.sh   # 在主节点上停止spark集群

/opt/spark/sbin/start-master.sh # 在主节点启动Master
/opt/spark/sbin/stop-master.sh  # 在主节点停止Master

/opt/spark/sbin/start-slaves.sh # 在从节点启动slaves
/opt/spark/sbin/stop-slaves.sh  # 在从节点停止slaves
```

?> 因为hadoop配置了全局变量，`start-all.sh`变成hadoop的启停命令，因此spark命令需要使用`./`或者绝对路径来执行

2. 验证集群

```bash
jps
```

- 任务运行web-ui界面端口：http://node01:4040/
- spark集群web-ui界面端口：http://node01:8080/
- spark提交任务时的通信端口：spark://node01:7077/

#### 1.2.6 客户端测试

1. 启动spark-shell

```bash
/opt/spark/bin/spark-shell --master spark://node01:7077
```

2. hadoop上传文件

```bash
hadoop fs -mkdir -p /test/input     # 创建目录
hadoop fs -put /opt/words.txt /test/input/words.txt     # 上传
hadoop fs -rm -r /test      # 测试后删除 
```

3. 执行

```bash
val textFile = sc.textFile("hdfs://node01:8020/test/input/words.txt")
val counts = textFile.flatMap(_.split(" ")).map((_,1)).reduceByKey(_ + _)
counts.collect
counts.saveAsTextFile("hdfs://node01:8020/test/output2022")
```

4. 查看结果

http://node01:9870/explorer.html#/test/output2022

### 1.3 StandAlone-HA模式

#### 1.3.1 集群规划

```lua
node01  master
node02  master worker/slave
node03  worker/slave
```

#### 1.3.2 启动zk

见[Zookeeper 3.7.0](deploy/zookeeper)搭建

```bash
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh start   # 每个节点分别启动
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh status  # 检测节点状态
```

#### 1.3.3 配置workers

```bash
cd /opt/spark/conf
cp workers.template workers
vim workers
# 添加内容
node02
node03
```

#### 1.3.4 配置master

```bash
cd /opt/spark/conf
cp spark-env.sh.template spark-env.sh
vim spark-env.sh
```

```sh
## 设置JAVA安装目录
JAVA_HOME=/usr/local/jdk1.8.0_202

## HADOOP软件配置文件目录，读取HDFS上文件和运行Spark在YARN集群时需要,先提前配上
HADOOP_CONF_DIR=/opt/hadoop-3.1.4/etc/hadoop
YARN_CONF_DIR=/opt/hadoop-3.1.4/etc/hadoop

## 指定提交任务的通信端口，这里无需要指定Master的IP，master通过zk选举得到
SPARK_MASTER_PORT=7077

SPARK_MASTER_WEBUI_PORT=8080

SPARK_WORKER_CORES=1
SPARK_WORKER_MEMORY=1g

## Zookeeper配置
SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=node01:2181,node02:2181,node03:2181 -Dspark.deploy.zookeeper.dir=/spark-ha"
```

#### 1.3.5 分发安装包

```bash
cd /opt/spark/
scp -r /opt/spark root@node02:$PWD
scp -r /opt/spark root@node03:$PWD
```

#### 1.3.6 启动集群

1. 启停命令

```bash
/opt/spark/sbin/start-all.sh        # 在主节点上启动spark集群
/opt/spark/sbin/start-master.sh     # 在node02节点启动第2个Master
```

2. 验证Master

```bash
jps
```

- node01节点web-ui界面端口：http://node01:8080/
- node02节点web-ui界面端口：http://node02:8080/

#### 1.3.7 客户端测试

1. 启动spark-shell

```bash
/opt/spark/bin/spark-shell --master spark://node01:7077,node02:7077
```

2. hadoop上传文件

```bash
hadoop fs -mkdir -p /test/input     # 创建目录
hadoop fs -put /opt/words.txt /test/input/words.txt     # 上传
hadoop fs -rm -r /test      # 测试后删除 
```

3. 执行

```bash
val textFile = sc.textFile("hdfs://node01:8020/test/input/words.txt")
val counts = textFile.flatMap(_.split(" ")).map((_,1)).reduceByKey(_ + _)
counts.collect
counts.saveAsTextFile("hdfs://node01:8020/test/output2022")
```

4. 查看结果

http://node01:9870/explorer.html#/test/output2022


### 1.4 On-Yarn模式

#### 1.4.1 HDFS修改配置

1. 关闭内存检查

```bash
vi /opt/hadoop-3.1.4/etc/hadoop/yarn-site.xml
```

```xml
<configuration>
    <!-- 配置yarn主节点的位置 -->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>node01.xuzhihao.net</value>
    </property>
    <!-- NodeManager上运行的附属服务。需配置成mapreduce_shuffle,才可运行MR程序。-->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <!-- 每个容器请求的最小内存资源（以MB为单位）。-->
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>512</value>
    </property>
    <!-- 每个容器请求的最大内存资源（以MB为单位）。-->
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>2048</value>
    </property>
    <!-- 容器虚拟内存与物理内存之间的比率。-->
    <property>
        <name>yarn.nodemanager.vmem-pmem-ratio</name>
        <value>4</value>
    </property>
    <!-- 开启日志聚合功能 -->
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
    <!-- 设置聚合日志在hdfs上的保存时间 -->
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>604800</value>
    </property>
    <!-- 设置yarn历史服务器地址 -->
    <property>
        <name>yarn.log.server.url</name>
        <value>http://node01:19888/jobhistory/logs</value>
    </property>
    <!-- 关闭yarn内存检查 -->
    <property>
        <name>yarn.nodemanager.pmem-check-enabled</name>
        <value>false</value>
    </property>
    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>
</configuration>
```

#### 1.4.2 Yarn配置分发

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
scp -r yarn-site.xml root@node02:$PWD
scp -r yarn-site.xml root@node03:$PWD
```

#### 1.4.3 启动HDFS

```bash
start-all.sh
```

#### 1.4.4 配置Spark的历史服务器和Yarn的整合

1. 修改spark-defaults.conf

```bash
cd /opt/spark/conf
cp spark-defaults.conf.template spark-defaults.conf
vi spark-defaults.conf
```

添加内容：

```conf
spark.eventLog.enabled                  true
spark.eventLog.dir                      hdfs://node01:8020/sparklog/
spark.eventLog.compress                 true
spark.yarn.historyServer.address        node01:18080
```

2. 修改spark-env.sh

```bash
cd /opt/spark/conf
vi spark-env.sh
```

```bash
## 配置spark历史日志存储地址
SPARK_HISTORY_OPTS="-Dspark.history.fs.logDirectory=hdfs://node01:8020/sparklog/ -Dspark.history.fs.cleaner.enabled=true"
```

3. 修改日志级别

```bash
cd /opt/spark/conf
cp log4j.properties.template log4j.properties
vim log4j.properties
# 将log4j.rootCategory=INFO, console修改为
log4j.rootCategory=WARN, console
```

#### 1.4.5 Spark配置分发

```bash
cd /opt/spark/conf
scp -r spark-env.sh root@node02:$PWD
scp -r spark-env.sh root@node03:$PWD
scp -r spark-defaults.conf root@node02:$PWD
scp -r spark-defaults.conf root@node03:$PWD
scp -r log4j.properties root@node02:$PWD
scp -r log4j.properties root@node03:$PWD
```

#### 1.4.6 配置依赖的Spark的jar包

1. 创建目录

```bash
hadoop fs -mkdir -p /sparklog       # 创建日志目录
hadoop fs -mkdir -p /spark/jars/    # 创建jar包目录
```

2. 上传jar包

```bash
hadoop fs -put /opt/spark/jars/* /spark/jars/
```

3. 在node1上修改spark-defaults.conf

```bash
vi /opt/spark/conf/spark-defaults.conf
#添加内容
spark.yarn.jars  hdfs://node01:8020/spark/jars/*
```

4. 分发同步

```
cd /opt/spark/conf
scp -r spark-defaults.conf root@node02:$PWD
scp -r spark-defaults.conf root@node03:$PWD
```

#### 1.4.7 启动服务

```bash
# 在node01执行命令
mr-jobhistory-daemon.sh start historyserver     # 启动MRHistoryServer
/opt/spark/sbin/start-history-server.sh         # 启动Spark HistoryServer
```

验证服务

- MRHistoryServer服务WEB UI页面：http://node01:19888
- Spark HistoryServer服务WEB UI页面：http://node01:18080/

#### 1.4.8 客户端测试

1. client模式

```bash
SPARK_HOME=/opt/spark
${SPARK_HOME}/bin/spark-submit \
--master yarn  \
--deploy-mode client \
--driver-memory 512m \
--driver-cores 1 \
--executor-memory 512m \
--num-executors 2 \
--executor-cores 1 \
--class org.apache.spark.examples.SparkPi \
${SPARK_HOME}/examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

验证：http://node01:8088/cluster ，点击`history`进入：http://node01:18080/history 的历史页面


2. cluster模式

```bash
SPARK_HOME=/opt/spark
${SPARK_HOME}/bin/spark-submit \
--master yarn \
--deploy-mode cluster \
--driver-memory 512m \
--executor-memory 512m \
--num-executors 1 \
--class org.apache.spark.examples.SparkPi \
${SPARK_HOME}/examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

验证：http://node01:8088/cluster ，点击`id`后的`logs`进入：http://node01:19888/jobhistory/logs 的历史页面


#### 1.4.9 自定义jar开发

参考代码：https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples

代码地址：https://github.com/xzh-net/scala/tree/main/spark3-test

```bash
mvn clean package   # 编译后上传
```

提交任务

```bash
SPARK_HOME=/opt/spark
${SPARK_HOME}/bin/spark-submit \
--master yarn \
--deploy-mode cluster \
--driver-memory 512m \
--executor-memory 512m \
--num-executors 1 \
--class net.xzh.test.WordCountOnYarn \
${SPARK_HOME}/examples/jars/original-spark3-test-1.0-SNAPSHOT.jar \
hdfs://node01:8020/test/input/words.txt \
hdfs://node01:8020/test/output2022
```