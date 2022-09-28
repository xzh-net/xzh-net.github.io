# Flume 1.9.0

Flume 是 Cloudera 提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统。Flume 基于流式架构，灵活简单

官网地址：http://flume.apache.org/

文档查看地址：http://flume.apache.org/FlumeUserGuide.html

下载地址：http://archive.apache.org/dist/flume/

## 1. 安装


### 1.1 上传解压

```bash
mkdir -p /opt/software
cd /opt/software
tar -zxvf apache-flume-1.9.0-bin.tar.gz -C /opt
mv /opt/apache-flume-1.9.0-bin/ /opt/flume
```

### 1.2 修改配置

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

### 1.3 配置环境变量

```bash
vim /etc/profile.d/my_env.sh
# 
export JAVA_HOME=/usr/local/jdk1.8.0_202
export HADOOP_HOME=/opt/hadoop-3.1.4
PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export PATH JAVA_HOME HADOOP_HOME
```

```bash
source /etc/profile
```

## 2. 案例

### 2.1 监控端口数据

#### 2.1.1 创建flume-netcat-logger.conf文件

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

#### 2.1.2 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/flume-netcat-logger.conf -Dflume.root.logger=INFO,console
# 简写
bin/flume-ng agent -c conf/ -n a1 -f job/flume-netcat-logger.conf -Dflume.root.logger=INFO,console
```

#### 2.1.3 发送数据

```bash
sudo yum install -y nc  # 安装
sudo netstat -nlp | grep 44444  # 检测端口占用
nc localhost 44444      # 发送数据
```

### 2.2 实时监控单个文件写入HDFS

#### 2.2.1 创建flume-file-hdfs.conf文件

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

#### 2.2.2 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a2 --conf-file job/flume-file-hdfs.conf
```

#### 2.2.3 开启Hadoop和Hive并操作Hive产生日志

```bash
start-all.sh                # 开启hadoop
cd /opt/apache-hive-3.1.2   # 启动hive
bin/hive
```

#### 2.2.4 查看HDFS上的数据

访问地址：http://node01:9870/


### 2.3 实时监控目录下多个新文件


#### 2.3.1 创建flume-dir-hdfs.conf文件

```bash
cd /opt/flume/job
vim flume-dir-hdfs.conf
```

```conf
a3.sources = r3
a3.sinks = k3
a3.channels = c3
# Describe/configure the source
a3.sources.r3.type = spooldir
a3.sources.r3.spoolDir = /opt/flume/upload
a3.sources.r3.fileSuffix = .COMPLETED
a3.sources.r3.fileHeader = true
#忽略所有以.tmp 结尾的文件，不上传
a3.sources.r3.ignorePattern = ([^ ]*\.tmp)
# Describe the sink
a3.sinks.k3.type = hdfs
a3.sinks.k3.hdfs.path =hdfs://node01:8020/flume/upload/%Y%m%d/%H
#上传文件的前缀
a3.sinks.k3.hdfs.filePrefix = upload-
#是否按照时间滚动文件夹
a3.sinks.k3.hdfs.round = true
#多少时间单位创建一个新的文件夹
a3.sinks.k3.hdfs.roundValue = 1
#重新定义时间单位
a3.sinks.k3.hdfs.roundUnit = hour
#是否使用本地时间戳
a3.sinks.k3.hdfs.useLocalTimeStamp = true
#积攒多少个 Event 才 flush 到 HDFS 一次
a3.sinks.k3.hdfs.batchSize = 100
#设置文件类型，可支持压缩
a3.sinks.k3.hdfs.fileType = DataStream
#多久生成一个新的文件
a3.sinks.k3.hdfs.rollInterval = 60
#设置每个文件的滚动大小大概是 128M
a3.sinks.k3.hdfs.rollSize = 134217700
#文件的滚动与 Event 数量无关
a3.sinks.k3.hdfs.rollCount = 0
# Use a channel which buffers events in memory
a3.channels.c3.type = memory
a3.channels.c3.capacity = 1000
a3.channels.c3.transactionCapacity = 100
# Bind the source and sink to the channel
a3.sources.r3.channels = c3
a3.sinks.k3.channel = c3
```

#### 2.3.2 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a3 --conf-file job/flume-dir-hdfs.conf
```

#### 2.3.3 向upload文件夹中添加文件

!> 在使用Spooling Directory Source时，不要在监控目录中创建并持续修改文件；上传完成的文件会以.COMPLETED 结尾；被监控文件夹每500毫秒扫描一次文件变动。

```bash
vi /opt/words.txt
cp /opt/words.txt /opt/flume/upload/
```

#### 2.3.4 查看HDFS上的数据

访问地址：http://node01:9870/

### 2.4 实时监控目录下的多个追加文件

#### 2.4.1 创建flume-taildir-hdfs.conf文件

```bash
cd /opt/flume/job
vim flume-taildir-hdfs.conf
```

```conf
a3.sources = r3
a3.sinks = k3
a3.channels = c3
# Describe/configure the source
a3.sources.r3.type = TAILDIR
a3.sources.r3.positionFile = /opt/flume/tail_dir.json
a3.sources.r3.filegroups = f1 f2
a3.sources.r3.filegroups.f1 = /opt/flume/files/.*file.*
a3.sources.r3.filegroups.f2 = /opt/flume/files2/.*log.*
# Describe the sink
a3.sinks.k3.type = hdfs
a3.sinks.k3.hdfs.path =hdfs://node01:8020/flume/upload2/%Y%m%d/%H
#上传文件的前缀
a3.sinks.k3.hdfs.filePrefix = upload-
#是否按照时间滚动文件夹
a3.sinks.k3.hdfs.round = true
#多少时间单位创建一个新的文件夹
a3.sinks.k3.hdfs.roundValue = 1
#重新定义时间单位
a3.sinks.k3.hdfs.roundUnit = hour
#是否使用本地时间戳
a3.sinks.k3.hdfs.useLocalTimeStamp = true
#积攒多少个 Event 才 flush 到 HDFS 一次
a3.sinks.k3.hdfs.batchSize = 100
#设置文件类型，可支持压缩
a3.sinks.k3.hdfs.fileType = DataStream
#多久生成一个新的文件
a3.sinks.k3.hdfs.rollInterval = 60
#设置每个文件的滚动大小大概是 128M
a3.sinks.k3.hdfs.rollSize = 134217700
#文件的滚动与 Event 数量无关
a3.sinks.k3.hdfs.rollCount = 0
# Use a channel which buffers events in memory
a3.channels.c3.type = memory
a3.channels.c3.capacity = 1000
a3.channels.c3.transactionCapacity = 100
# Bind the source and sink to the channel
a3.sources.r3.channels = c3
a3.sinks.k3.channel = c3
```

#### 2.4.2 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a3 --conf-file job/flume-taildir-hdfs.conf
```

#### 2.4.3 写入文件

```bash
cd /opt/flume/
mkdir files files2
cd /opt/flume/files 
echo file1 >> file1.txt
echo file2 >> file2.txt
# 每个文件夹组下的文件对应一个hdfs文件
cd /opt/flume/files2 
echo log2 >> log2.txt
```

#### 2.4.4 查看HDFS上的数据

访问地址：http://node01:9870/
 

### 2.5 采集数据写入pulsar

代码地址：https://github.com/xzh-net/jakarta-learn/tree/main/pulsar-flume-ng-sink

#### 2.5.1 编译上传

```bash
cp /opt/software/flume-ng-pulsar-sink-1.9.0.jar /opt/flume/lib/
```

#### 2.5.2 创建flume-netcat-pulsar.conf文件

```bash
cd /opt/flume/job
vim flume-netcat-pulsar.conf
```

```conf
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# Describe the sink
a1.sinks.k1.type = org.apache.flume.sink.pulsar.PulsarSink
a1.sinks.k1.serviceUrl = node01.xuzhihao.net:6650,node02.xuzhihao.net:6650,node03.xuzhihao.net:6650
a1.sinks.k1.topicName = test
a1.sinks.k1.producerName = testProducer
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 1000

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

#### 2.5.3 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/flume-netcat-pulsar.conf
```

#### 2.5.4 发送数据

```bash
nc localhost 44444
```

#### 2.5.5 监听数据

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-client consume persistent://public/default/test -s "consumer-test"  
```

### 2.6 复制

#### 2.6.1 需求

使用Flume-1监控文件变动，Flume-1将变动内容传递给Flume-2，Flume-2负责存储到HDFS。同时Flume-1将变动内容传递给Flume-3，Flume-3负责输出到Local FileSystem

#### 2.6.2 创建flume-file-flume.conf

```bash
cd /opt/flume/job/group1
vim flume-file-flume.conf
```

```conf
# Name the components on this agent
a1.sources = r1
a1.sinks = k1 k2
a1.channels = c1 c2
# 将数据流复制给所有 channel
a1.sources.r1.selector.type = replicating
# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /opt/apache-hive-3.1.2/logs/hive.log
a1.sources.r1.shell = /bin/bash -c
# Describe the sink
# sink 端的 avro 是一个数据发送者
a1.sinks.k1.type = avro
a1.sinks.k1.hostname = node01
a1.sinks.k1.port = 4141
a1.sinks.k2.type = avro
a1.sinks.k2.hostname = node01
a1.sinks.k2.port = 4142
# Describe the channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
a1.channels.c2.type = memory
a1.channels.c2.capacity = 1000
a1.channels.c2.transactionCapacity = 100
# Bind the source and sink to the channel
a1.sources.r1.channels = c1 c2
a1.sinks.k1.channel = c1
a1.sinks.k2.channel = c2
```

#### 2.6.3 创建flume-flume-hdfs.conf

```bash
cd /opt/flume/job/group1
vim flume-flume-hdfs.conf
```

```conf
# Name the components on this agent
a2.sources = r1
a2.sinks = k1
a2.channels = c1
# Describe/configure the source
# source 端的 avro 是一个数据接收服务
a2.sources.r1.type = avro
a2.sources.r1.bind = node01
a2.sources.r1.port = 4141
# Describe the sink
a2.sinks.k1.type = hdfs
a2.sinks.k1.hdfs.path = hdfs://node01:8020/flume2/%Y%m%d/%H
#上传文件的前缀
a2.sinks.k1.hdfs.filePrefix = flume2-
#是否按照时间滚动文件夹
a2.sinks.k1.hdfs.round = true
#多少时间单位创建一个新的文件夹
a2.sinks.k1.hdfs.roundValue = 1
#重新定义时间单位
a2.sinks.k1.hdfs.roundUnit = hour
#是否使用本地时间戳
a2.sinks.k1.hdfs.useLocalTimeStamp = true
#积攒多少个 Event 才 flush 到 HDFS 一次
a2.sinks.k1.hdfs.batchSize = 100
#设置文件类型，可支持压缩
a2.sinks.k1.hdfs.fileType = DataStream
#多久生成一个新的文件
a2.sinks.k1.hdfs.rollInterval = 30
#设置每个文件的滚动大小大概是 128M
a2.sinks.k1.hdfs.rollSize = 134217700
#文件的滚动与 Event 数量无关
a2.sinks.k1.hdfs.rollCount = 0
# Describe the channel
a2.channels.c1.type = memory
a2.channels.c1.capacity = 1000
a2.channels.c1.transactionCapacity = 100
# Bind the source and sink to the channel
a2.sources.r1.channels = c1
a2.sinks.k1.channel = c1
```

#### 2.6.4 创建flume-flume-dir.conf

```bash
cd /opt/flume/job/group1
vim flume-flume-dir.conf
```

```conf
# Name the components on this agent
a3.sources = r1
a3.sinks = k1
a3.channels = c2
# Describe/configure the source
a3.sources.r1.type = avro
a3.sources.r1.bind = node01
a3.sources.r1.port = 4142
# Describe the sink
a3.sinks.k1.type = file_roll
a3.sinks.k1.sink.directory = /opt/flume/data/flume3
# Describe the channel
a3.channels.c2.type = memory
a3.channels.c2.capacity = 1000
a3.channels.c2.transactionCapacity = 100
# Bind the source and sink to the channel
a3.sources.r1.channels = c2
a3.sinks.k1.channel = c2
```

!> 输出的本地目录必须是已经存在的目录，如果该目录不存在，并不会创建新的目录

```bash
mkdir -p /opt/flume/data/flume3
```

#### 2.6.5 开启Hadoop和Hive并操作Hive产生日志

```bash
start-all.sh                # 开启hadoop
cd /opt/apache-hive-3.1.2   # 启动hive
bin/hive
```

#### 2.6.6 启动flume

avro通信框架需要先启动服务端，所以需要先启动source端

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a3 --conf-file job/group1/flume-flume-dir.conf
bin/flume-ng agent --conf conf/ --name a2 --conf-file job/group1/flume-flume-hdfs.conf
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/group1/flume-file-flume.conf
```

#### 2.6.7 查看HDFS上的数据

访问地址：http://node01:9870/

#### 2.6.8 查看本地数据

```bash
ll /opt/flume/data/flume3
```

### 2.7 多路复用

#### 2.7.1 需求

使用Flume采集服务器本地日志，需要按照日志类型的不同，将不同种类的日志发往不同的分析系统，需要自定义拦截器

#### 2.7.2 自定义拦截器

代码地址：https://github.com/xzh-net/jakarta-learn/tree/main/flume

#### 2.7.2 编译上传

```bash
cp /opt/software/flume-test-1.0-SNAPSHOT.jar /opt/flume/lib/
```

#### 2.7.3 node01创建flume1.conf

```bash
cd /opt/flume/job/group4
vim flume1.conf
```

```conf
# Name the components on this agent
a1.sources = r1
a1.sinks = k1 k2
a1.channels = c1 c2
# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type =net.xzh.flume.interceptor.MyInterceptor$Builder
a1.sources.r1.selector.type = multiplexing
a1.sources.r1.selector.header = from
a1.sources.r1.selector.mapping.xzh = c1
a1.sources.r1.selector.mapping.other = c2
# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.hostname = node02
a1.sinks.k1.port = 4141
a1.sinks.k2.type=avro
a1.sinks.k2.hostname = node03
a1.sinks.k2.port = 4242
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
# Use a channel which buffers events in memory
a1.channels.c2.type = memory
a1.channels.c2.capacity = 1000
a1.channels.c2.transactionCapacity = 100
# Bind the source and sink to the channel
a1.sources.r1.channels = c1 c2
a1.sinks.k1.channel = c1
a1.sinks.k2.channel = c2
```

#### 2.7.4 node02创建flume2.conf

```bash
cd /opt/flume/job/group4
vim flume2.conf
```

```conf
a2.sources = r1
a2.sinks = k1
a2.channels = c1
a2.sources.r1.type = avro
a2.sources.r1.bind = node02
a2.sources.r1.port = 4141
a2.sinks.k1.type = logger
a2.channels.c1.type = memory
a2.channels.c1.capacity = 1000
a2.channels.c1.transactionCapacity = 100
a2.sinks.k1.channel = c1
a2.sources.r1.channels = c1
```

#### 2.7.5 node03创建flume3.conf

```bash
cd /opt/flume/job/group4
vim flume3.conf
```

```conf
a3.sources = r1
a3.sinks = k1
a3.channels = c1
a3.sources.r1.type = avro
a3.sources.r1.bind = node03
a3.sources.r1.port = 4242
a3.sinks.k1.type = logger
a3.channels.c1.type = memory
a3.channels.c1.capacity = 1000
a3.channels.c1.transactionCapacity = 100
a3.sinks.k1.channel = c1
a3.sources.r1.channels = c1
```

#### 2.7.6 启动flume

avro通信框架需要先启动服务端，所以需要先启动source端

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a3 --conf-file job/group4/flume3.conf -Dflume.root.logger=INFO,console  # node03执行
bin/flume-ng agent --conf conf/ --name a2 --conf-file job/group4/flume2.conf -Dflume.root.logger=INFO,console  # node02执行
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/group4/flume1.conf -Dflume.root.logger=INFO,console  # node01执行
```

#### 2.7.7 发送数据

```bash
nc localhost 44444   # node01执行，输入xzh
```

#### 2.7.8 查看数据

node02，node03查看控制台数据

### 2.8 负载均衡和故障转移

#### 2.8.1 需求

使用Flume1监控一个端口，其sink组中的sink分别对接Flume2和Flume3，采用FailoverSinkProcessor，实现故障转移的功能

#### 2.8.2 创建flume-netcat-flume.conf

```bash
cd /opt/flume/job/group2
vim flume-netcat-flume.conf
```

```conf
# Name the components on this agent
a1.sources = r1
a1.channels = c1
a1.sinkgroups = g1
a1.sinks = k1 k2
# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444
a1.sinkgroups.g1.processor.type = failover
a1.sinkgroups.g1.processor.priority.k1 = 5
a1.sinkgroups.g1.processor.priority.k2 = 10
a1.sinkgroups.g1.processor.maxpenalty = 10000
# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.hostname = node01
a1.sinks.k1.port = 4141
a1.sinks.k2.type = avro
a1.sinks.k2.hostname = node01
a1.sinks.k2.port = 4142
# Describe the channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinkgroups.g1.sinks = k1 k2
a1.sinks.k1.channel = c1
a1.sinks.k2.channel = c1
```

#### 2.8.3 创建flume-flume-console1.conf

```bash
cd /opt/flume/job/group2
vim flume-flume-console1.conf
```

```conf
# Name the components on this agent
a2.sources = r1
a2.sinks = k1
a2.channels = c1
# Describe/configure the source
a2.sources.r1.type = avro
a2.sources.r1.bind = node01
a2.sources.r1.port = 4141
# Describe the sink
a2.sinks.k1.type = logger
# Describe the channel
a2.channels.c1.type = memory
a2.channels.c1.capacity = 1000
a2.channels.c1.transactionCapacity = 100
# Bind the source and sink to the channel
a2.sources.r1.channels = c1
a2.sinks.k1.channel = c1
```

#### 2.8.4 创建flume-flume-console2.conf

```bash
cd /opt/flume/job/group2
vim flume-flume-console2.conf
```

```conf
# Name the components on this agent
a3.sources = r1
a3.sinks = k1
a3.channels = c2
# Describe/configure the source
a3.sources.r1.type = avro
a3.sources.r1.bind = node01
a3.sources.r1.port = 4142
# Describe the sink
a3.sinks.k1.type = logger
# Describe the channel
a3.channels.c2.type = memory
a3.channels.c2.capacity = 1000
a3.channels.c2.transactionCapacity = 100
# Bind the source and sink to the channel
a3.sources.r1.channels = c2
a3.sinks.k1.channel = c2
```

#### 2.8.5 启动flume

avro通信框架需要先启动服务端，所以需要先启动source端

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a3 --conf-file job/group2/flume-flume-console2.conf -Dflume.root.logger=INFO,console
bin/flume-ng agent --conf conf/ --name a2 --conf-file job/group2/flume-flume-console1.conf -Dflume.root.logger=INFO,console
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/group2/flume-netcat-flume.conf
```

#### 2.8.6 发送数据

```bash
nc localhost 44444
```

#### 2.8.7 查看Flume2和Flume3日志

模拟异常kill掉Flume2进程，观察Flume3控制台日志

#### 2.8.8 负载均衡

修改flume-netcat-flume.conf

```conf
a1.sinkgroups.g1.processor.type = failover
a1.sinkgroups.g1.processor.priority.k1 = 5
a1.sinkgroups.g1.processor.priority.k2 = 10
a1.sinkgroups.g1.processor.maxpenalty = 10000
```

替换为

```conf
a1.sinkgroups.g1.processor.type = load_balance
a1.sinkgroups.g1.processor.backoff = true
a1.sinkgroups.g1.processor.selector = random
a1.sinkgroups.g1.processor.selector.maxTimeOut = 30000 # 退避算法最大值
```

### 2.9 聚合

#### 2.9.1 需求

- node01上的Flume-1监控文件/opt/apache-hive-3.1.2/logs/hive.log
- node02上的Flume-2监控某一个端口的数据流
- Flume-1与Flume-2将数据发送给node03上的Flume-3，Flume-3将最终数据打印到控制台

#### 2.9.2 Flume分发

```bash
cd /opt/flume
scp -r /opt/flume root@node02:$PWD
scp -r /opt/flume root@node03:$PWD
```

#### 2.9.3 node01创建flume1-logger-flume.conf

```bash
cd /opt/flume/job/group3
vim flume1-logger-flume.conf
```

```conf
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1
# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /opt/flume/data/group3.log
a1.sources.r1.shell = /bin/bash -c
# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.hostname = node03
a1.sinks.k1.port = 4141
# Describe the channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

#### 2.9.4 node02创建flume2-netcat-flume.conf

```bash
cd /opt/flume/job/group3
vim flume2-netcat-flume.conf
```

```conf
# Name the components on this agent
a2.sources = r1
a2.sinks = k1
a2.channels = c1
# Describe/configure the source
a2.sources.r1.type = netcat
a2.sources.r1.bind = node02
a2.sources.r1.port = 44444
# Describe the sink
a2.sinks.k1.type = avro
a2.sinks.k1.hostname = node03
a2.sinks.k1.port = 4141
# Use a channel which buffers events in memory
a2.channels.c1.type = memory
a2.channels.c1.capacity = 1000
a2.channels.c1.transactionCapacity = 100
# Bind the source and sink to the channel
a2.sources.r1.channels = c1
a2.sinks.k1.channel = c1
```


#### 2.9.5 node03创建flume3-flume-logger.conf

```bash
cd /opt/flume/job/group3
vim flume3-flume-logger.conf
```

```conf
# Name the components on this agent
a3.sources = r1
a3.sinks = k1
a3.channels = c1
# Describe/configure the source
a3.sources.r1.type = avro
a3.sources.r1.bind = node03
a3.sources.r1.port = 4141
# Describe the sink
a3.sinks.k1.type = logger
# Describe the channel
a3.channels.c1.type = memory
a3.channels.c1.capacity = 1000
a3.channels.c1.transactionCapacity = 100
# Bind the source and sink to the channel
a3.sources.r1.channels = c1
a3.sinks.k1.channel = c1
```

#### 2.9.6 启动flume

avro通信框架需要先启动服务端，所以需要先启动source端

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a3 --conf-file job/group3/flume3-flume-logger.conf -Dflume.root.logger=INFO,console  # node03执行
bin/flume-ng agent --conf conf/ --name a2 --conf-file job/group3/flume2-netcat-flume.conf                                   # node02执行
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/group3/flume1-logger-flume.conf                                   # node01执行
```

#### 2.9.7 发送数据

```bash
echo 'hello' > /opt/flume/data/group3.log   # node01执行
nc localhost 44444                          # node02执行
```

#### 2.9.8 查看数据

node03查看控制台数据

### 2.10 自定义source

#### 2.10.1 需求

使用flume接收数据，并给每条数据添加前缀，输出到控制台。前缀可从flume配置文件中配置

#### 2.10.2 代码编写

代码地址：https://github.com/xzh-net/jakarta-learn/tree/main/flume

#### 2.10.3 编译上传

```bash
cp /opt/software/flume-test-1.0-SNAPSHOT.jar /opt/flume/lib/
```

#### 2.10.4 创建flume1.conf

```bash
cd /opt/flume/job/group5
vim flume1.conf
```

```conf
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1
# Describe/configure the source
a1.sources.r1.type = net.xzh.flume.source.MySource
a1.sources.r1.delay = 2000
a1.sources.r1.prefix = start-
a1.sources.r1.subfix = -end
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

#### 2.10.5 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/group5/flume1.conf -Dflume.root.logger=INFO,console
```

### 2.11 自定义sink

#### 2.11.1 需求

使用flume接收数据，并在Sink端给每条数据添加前缀和后缀，输出到控制台。前后缀可在flume任务配置文件中配置

#### 2.11.2 代码编写

代码地址：https://github.com/xzh-net/jakarta-learn/tree/main/flume

#### 2.11.3 编译上传

```bash
cp /opt/software/flume-test-1.0-SNAPSHOT.jar /opt/flume/lib/
```

#### 2.11.4 创建flume1.conf

```bash
cd /opt/flume/job/group6
vim flume1.conf
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
a1.sinks.k1.type = net.xzh.flume.sink.MySink
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

#### 2.11.5 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/group6/flume1.conf -Dflume.root.logger=INFO,console
```

#### 2.11.6 发送数据

```bash
nc localhost 44444
```

## 3. 源码

### 3.1 修改flume-taildir-source

问题描述：在数仓项目中，使用Flume的TairDir Source监控日志文件，当文件更名之后会重新读取该文件造成重复

解决办法：
  - 使用不更名打印日志框架（logback），每天会新生成一个日志文件，文件后面会加上当天的日期信息，所以不会重复
  - 修改源码，让TairDir Source判断文件时只看iNode的值

#### 3.1.1 修改TailFile

org.apache.flume.source.taildir.TailFile

```java
public boolean updatePos(String path, long inode, long pos) throws IOException {
    //update by xzh
    //if (this.inode == inode && this.path.equals(path)) {
    if (this.inode == inode)
      setPos(pos);
      updateFilePos(pos);
      logger.info("Updated position, file: " + path + ", inode: " + inode + ", pos: " + pos);
      return true;
    }
    return false;
  }
```

#### 3.1.2 修改ReliableTaildirEventReader

org.apache.flume.source.taildir.ReliableTaildirEventReader

```java
public List<Long> updateTailFiles(boolean skipToEnd) throws IOException {
    updateTime = System.currentTimeMillis();
    List<Long> updatedInodes = Lists.newArrayList();

    for (TaildirMatcher taildir : taildirCache) {
      Map<String, String> headers = headerTable.row(taildir.getFileGroup());

      for (File f : taildir.getMatchingFiles()) {
        long inode;
        try {
          inode = getInode(f);
        } catch (NoSuchFileException e) {
          logger.info("File has been deleted in the meantime: " + e.getMessage());
          continue;
        }
        TailFile tf = tailFiles.get(inode);
        //update by xzh
        //if (tf == null || !tf.getPath().equals(f.getAbsolutePath())) {
        if (tf == null)    
          long startPos = skipToEnd ? f.length() : 0;
          tf = openFile(f, headers, inode, startPos);
        } else {
          boolean updated = tf.getLastUpdated() < f.lastModified() || tf.getPos() != f.length();
          if (updated) {
            if (tf.getRaf() == null) {
              tf = openFile(f, headers, inode, tf.getPos());
            }
            if (f.length() < tf.getPos()) {
              logger.info("Pos " + tf.getPos() + " is larger than file size! "
                  + "Restarting from pos 0, file: " + tf.getPath() + ", inode: " + inode);
              tf.updatePos(tf.getPath(), inode, 0);
            }
          }
          tf.setNeedTail(updated);
        }
        tailFiles.put(inode, tf);
        updatedInodes.add(inode);
      }
    }
    return updatedInodes;
  }
```