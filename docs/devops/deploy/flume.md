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

1. 创建Flume Agent配置文件

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

2. 启动flume端口

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



### 1.2 集群