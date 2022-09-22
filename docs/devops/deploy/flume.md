# Flume 1.9.0

Flume 是 Cloudera 提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统。Flume 基于流式架构，灵活简单

官网地址：http://flume.apache.org/

文档查看地址：http://flume.apache.org/FlumeUserGuide.html

下载地址：http://archive.apache.org/dist/flume/

## 1. 安装

### 1.1 单机

#### 1.1.1 上传解压

```bash
mkdir -p /opt/software
cd /opt/software
tar -zxvf apache-flume-1.9.0-bin.tar.gz -C /opt
mv /opt/apache-flume-1.9.0-bin/ /opt/flume
```

#### 1.1.2 修改配置

```bash
cd /opt/flume/conf
cp flume-env.sh.template  flume-env.sh
vim flume-env.sh
# 修改
export JAVA_HOME=/usr/local/jdk1.8.0_202
```

将lib文件夹下的guava-11.0.2.jar删除以兼容Hadoop3.1.3

```bash
rm /opt/flume/lib/guava11.0.2.jar
cp /opt/hadoop-3.1.4/share/hadoop/common/lib/guava-27.0-jre.jar /opt/flume/lib/
```

#### 1.1.3 配置环境变量

```bash
vim /etc/profile.d/my_env.sh
# 
export JAVA_HOME=/usr/local/jdk1.8.0_202
export HADOOP_HOME=/opt/hadoop-3.1.4
PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export PATH JAVA_HOME HADOOP_HOME
```

#### 1.1.4 监控端口数据测试

1. 创建flume-netcat-logger.conf文件

```bash
cd /opt/flume/
mkdir job
cd job
vim flume-netcat-logger.conf
```

```conf
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1
# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444
# Describe the sink
a1.sinks.k1.type = logger
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

2. 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/flume-netcat-logger.conf -Dflume.root.logger=INFO,console
# 简写
bin/flume-ng agent -c conf/ -n a1 -f job/flume-netcat-logger.conf -Dflume.root.logger=INFO,console
```

3. 发送数据

```bash
sudo yum install -y nc  # 安装
sudo netstat -nlp | grep 44444  # 检测端口占用
nc localhost 44444      # 发送数据
```

#### 1.1.5 实时监控单个文件写入hdfs

1. 创建flume-file-hdfs.conf文件

```bash
cd /opt/flume/job
vim flume-file-hdfs.conf
```

```conf
# Name the components on this agent
a2.sources = r2
a2.sinks = k2
a2.channels = c2
# Describe/configure the source
a2.sources.r2.type = exec
a2.sources.r2.command = tail -F /opt/apache-hive-3.1.2/logs/hive.log
# Describe the sink
a2.sinks.k2.type = hdfs
a2.sinks.k2.hdfs.path = hdfs://node01:8020/flume/%Y%m%d/%H
#上传文件的前缀
a2.sinks.k2.hdfs.filePrefix = logs-
#是否按照时间滚动文件夹
a2.sinks.k2.hdfs.round = true
#多少时间单位创建一个新的文件夹
a2.sinks.k2.hdfs.roundValue = 1
#重新定义时间单位
a2.sinks.k2.hdfs.roundUnit = hour
#是否使用本地时间戳
a2.sinks.k2.hdfs.useLocalTimeStamp = true
#积攒多少个 Event 才 flush 到 HDFS 一次
a2.sinks.k2.hdfs.batchSize = 100
#设置文件类型，可支持压缩
a2.sinks.k2.hdfs.fileType = DataStream
#多久生成一个新的文件
a2.sinks.k2.hdfs.rollInterval = 60
#设置每个文件的滚动大小
a2.sinks.k2.hdfs.rollSize = 134217700
#文件的滚动与 Event 数量无关
a2.sinks.k2.hdfs.rollCount = 0
# Use a channel which buffers events in memory
a2.channels.c2.type = memory
a2.channels.c2.capacity = 1000
a2.channels.c2.transactionCapacity = 100
# Bind the source and sink to the channel
a2.sources.r2.channels = c2
a2.sinks.k2.channel = c2
```

2. 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a2 --conf-file job/flume-file-hdfs.conf
```

3. 开启Hadoop和Hive并操作Hive产生日志

```bash
start-all.sh                # 开启hadoop
cd /opt/apache-hive-3.1.2   # 启动hive
bin/hive
```


#### 1.1.6 实时监控单个追加文件

#### 1.1.7 实时监控单个追加文件
 
### 1.2 集群