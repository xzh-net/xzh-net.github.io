# Spark 3.1.3

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

#### 1.1.3 测试

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

!> 因为hadoop配置了全局变量，`start-all.sh`变成hadoop的启停命令，因此spark命令需要使用`./`或者绝对路径来执行

2. 验证集群

```bash
jps
```

- 任务运行web-ui界面端口：http://node01:4040/
- spark集群web-ui界面端口：http://node01:8080/
- spark提交任务时的通信端口：spark://node01:7077/

#### 1.2.6 测试

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

见[Zookeeper 3.7.0](devops/deploy/zookeeper)搭建

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

#### 1.3.7 测试

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