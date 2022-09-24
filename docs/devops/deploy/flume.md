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

#### 1.1.5 实时监控单个文件写入HDFS

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

4. 查看HDFS上的数据

访问地址：http://node01:9870/


#### 1.1.6 实时监控目录下多个新文件


1. 创建flume-dir-hdfs.conf文件

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

2. 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a3 --conf-file job/flume-dir-hdfs.conf
```

3. 向upload文件夹中添加文件

!> 在使用Spooling Directory Source时，不要在监控目录中创建并持续修改文件；上传完成的文件会以.COMPLETED 结尾；被监控文件夹每500毫秒扫描一次文件变动。

```bash
vi /opt/words.txt
cp /opt/words.txt /opt/flume/upload/
```

4. 查看HDFS上的数据

访问地址：http://node01:9870/

#### 1.1.7 实时监控目录下的多个追加文件

1. 创建flume-taildir-hdfs.conf文件

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

2. 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a3 --conf-file job/flume-taildir-hdfs.conf
```

3. 写入文件

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

4. 查看HDFS上的数据

访问地址：http://node01:9870/
 

#### 1.1.8 采集数据到pulsar

代码地址：https://github.com/xzh-net/jakarta-learn/tree/main/pulsar-flume-ng-sink

1. 上传jar

```bash
cp /opt/software/flume-ng-pulsar-sink-1.9.0.jar /opt/flume/lib/
```

2. 创建flume-netcat-pulsar.conf文件

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

3. 启动flume

```bash
cd /opt/flume/
bin/flume-ng agent --conf conf/ --name a1 --conf-file job/flume-netcat-pulsar.conf
```

4. 发送数据

```bash
nc localhost 44444
```

5. 监听数据

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-client consume persistent://public/default/test -s "consumer-test"  
```

### 1.2 集群

## 2. 源码

### 2.1 修改flume-taildir-source

问题描述：在数仓项目中，使用Flume的TairDir Source监控日志文件，当文件更名之后会重新读取该文件造成重复

解决办法：
  - 使用不更名打印日志框架（logback），每天会新生成一个日志文件，文件后面会加上当天的日期信息，所以不会重复
  - 修改源码，让TairDir Source判断文件时只看iNode的值

#### 2.1.1 修改TailFile

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

#### 2.1.2 修改ReliableTaildirEventReader

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