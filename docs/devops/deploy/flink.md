# Flink-1.12.0 scala_2.12

## 1. 安装

### 1.1 Local模式

#### 1.1.1 上传解压

```bash
mkdir -p /opt/software
cd /opt/software
tar -zxvf flink-1.12.0-bin-scala_2.12.tgz -C /opt
```

#### 1.1.2 修改权限

```bash
chown -R root:root /opt/flink-1.12.0
```

#### 1.1.3 启动服务

```bash
/opt/flink-1.12.0/bin/start-cluster.sh      # 启动服务
/opt/flink-1.12.0/bin/stop-cluster.sh       # 停止服务
```

#### 1.1.4 客户端测试

1. 创建测试文件

```bash
vim /opt/words.txt
```

```
hello me you her
hello me you
hello me
hello
```

2. 执行官方示例

```bash
/opt/flink-1.12.0/bin/flink run /opt/flink-1.12.0/examples/batch/WordCount.jar --input /opt/words.txt --output /opt/out
```

3. 启动flink交互式窗口

```bash
/opt/flink-1.12.0/bin/start-scala-shell.sh local
```

4. 执行

```bash
benv.readTextFile("/opt/words.txt").flatMap(_.split(" ")).map((_,1)).groupBy(0).sum(1).print()
```

!> 目前所有Scala2.12版本的安装包暂时都不支持 Scala Shell

5. Web UI

http://node01:8081/#/overview

### 1.2 StandAlone模式

#### 1.2.1 集群规划

```lua
node01  JobManager + TaskManager(Master + Slave)
node02  TaskManager(Slave)
node03  TaskManager(Slave)
```


#### 1.2.2 修改配置

```bash
vim /opt/flink-1.12.0/conf/flink-conf.yaml
```

```conf
# 编辑内容
jobmanager.rpc.address: node01
taskmanager.numberOfTaskSlots: 2
web.submit.enable: true

# 添加历史服务器
jobmanager.archive.fs.dir: hdfs://node01:8020/flink/completed-jobs/
historyserver.web.address: node01
historyserver.web.port: 8082
historyserver.archive.fs.dir: hdfs://node01:8020/flink/completed-jobs/
```

#### 1.2.3 配置masters

```bash
vim /opt/flink-1.12.0/conf/masters
# 添加内容
node01:8081
```

#### 1.2.4 配置slaves

```bash
vim /opt/flink-1.12.0/conf/workers
# 添加内容
node01
node02
node03
```

#### 1.2.5 配置环境变量

```bash
vim /etc/profile
export HADOOP_CONF_DIR=/opt/hadoop-3.1.4/etc/hadoop
# source /etc/profile
```

#### 1.2.6 分发安装包

```bash
cd /opt/flink-1.12.0
scp -r /opt/flink-1.12.0 root@node02:$PWD
scp -r /opt/flink-1.12.0 root@node03:$PWD
scp  /etc/profile node02:/etc/profile
scp  /etc/profile node03:/etc/profile
```

#### 1.2.7 启动集群

1. 启停命令

```bash
/opt/flink-1.12.0/bin/historyserver.sh start    # 启动历史服务器，启动之前创建路径hdfs://node01:8020/flink/completed-jobs/
/opt/flink-1.12.0/bin/start-cluster.sh          # 在主节点上启动flink集群
/opt/flink-1.12.0/bin/stop-cluster.sh           # 在主节点上停止flink集群

/opt/flink-1.12.0/bin/jobmanager.sh start       # 在主节点启动jobmanager
/opt/flink-1.12.0/bin/jobmanager.sh stop        # 在主节点停止jobmanager

/opt/flink-1.12.0/bin/taskmanager.sh start      # 在从节点启动taskmanager
/opt/flink-1.12.0/bin/taskmanager.sh stop       # 在从节点停止taskmanager
```

!> 整合hadoop3.1.4过程中会出现historyserver启动失败，下载jar放置到flink中lib文件夹下

下载地址：https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.7.5-10.0/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar

2. 验证集群

```bash
/opt/shell/xcall
```

- WebUI界面端口：http://node01:8081/#/overview
- 历史服务器端口：http://node01:8082/#/overview


#### 1.2.8 客户端测试

1. 执行官方示例

```bash
/opt/flink-1.12.0/bin/flink run /opt/flink-1.12.0/examples/batch/WordCount.jar
```

2. 查看结果

http://node01:8081/#/job/completed

### 1.3 StandAlone-HA模式

#### 1.3.1 集群规划

```lua
node01  JobManager + TaskManager(Master + Slave)
node02  JobManager + TaskManager(Master + Slave)
node03  TaskManager(Slave)
```

#### 1.3.2 启动zk

见[Zookeeper 3.7.0](devops/deploy/zookeeper)搭建

```bash
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh start   # 每个节点分别启动
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh status  # 检测节点状态
```

#### 1.3.3 启动HDFS

```bash
start-hdfs.sh
```

#### 1.3.4 修改配置

```bash
vim /opt/flink-1.12.0/conf/flink-conf.yaml
```

```conf
# 编辑内容
jobmanager.rpc.address: node01
taskmanager.numberOfTaskSlots: 2
web.submit.enable: true

# 添加历史服务器
jobmanager.archive.fs.dir: hdfs://node01:8020/flink/completed-jobs/
historyserver.web.address: node01
historyserver.web.port: 8082
historyserver.archive.fs.dir: hdfs://node01:8020/flink/completed-jobs/

# 添加HA
state.backend: filesystem
state.backend.fs.checkpointdir: hdfs://node01:8020/flink-checkpoints
high-availability: zookeeper
high-availability.storageDir: hdfs://node01:8020/flink/ha/
high-availability.zookeeper.quorum: node01:2181,node02:2181,node03:2181
```

#### 1.3.5 配置masters

```bash
vim /opt/flink-1.12.0/conf/masters
# 添加内容
node01:8081
node02:8081
```

#### 1.3.6 配置slaves

```bash
vim /opt/flink-1.12.0/conf/workers
# 添加内容
node01
node02
node03
```

#### 1.3.7 配置环境变量

```bash
vim /etc/profile
export HADOOP_CONF_DIR=/opt/hadoop-3.1.4/etc/hadoop
# source /etc/profile
```

#### 1.3.8 分发安装包

```bash
cd /opt/flink-1.12.0
scp -r /opt/flink-1.12.0 root@node02:$PWD
scp -r /opt/flink-1.12.0 root@node03:$PWD
scp  /etc/profile node02:/etc/profile
scp  /etc/profile node03:/etc/profile
```

#### 1.3.9 修改配置node02

```bash
vim /opt/flink-1.12.0/conf/flink-conf.yaml
```

```conf
# 编辑内容
jobmanager.rpc.address: node02
```

#### 1.3.10 启动集群

1. 启停命令

```bash
/opt/flink-1.12.0/bin/historyserver.sh start    # 启动历史服务器，启动之前创建路径hdfs://node01:8020/flink/completed-jobs/
/opt/flink-1.12.0/bin/start-cluster.sh          # 在主节点上启动flink集群
/opt/flink-1.12.0/bin/stop-cluster.sh           # 在主节点上停止flink集群

/opt/flink-1.12.0/bin/jobmanager.sh start       # 在主节点启动jobmanager
/opt/flink-1.12.0/bin/jobmanager.sh stop        # 在主节点停止jobmanager

/opt/flink-1.12.0/bin/taskmanager.sh start      # 在从节点启动taskmanager
/opt/flink-1.12.0/bin/taskmanager.sh stop       # 在从节点停止taskmanager
```

!> 整合hadoop3.1.4过程中会出现historyserver启动失败，下载jar放置到flink中lib文件夹下

下载地址：https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.7.5-10.0/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar

2. 验证集群

```bash
/opt/shell/xcall
```

3. 访问WebUI

- http://node01:8081/#/job-manager/config
- http://node02:8081/#/job-manager/config


#### 1.3.11 客户端测试

1. 执行官方示例

```bash
/opt/flink-1.12.0/bin/flink run /opt/flink-1.12.0/examples/batch/WordCount.jar
```

kill 掉node01的`StandaloneSessionClusterEntrypoint`以后再次执行，仍然可以通过`http://node02:8081/#/job/completed`看到执行结果，说明HA生效




### 1.4 On-Yarn模式