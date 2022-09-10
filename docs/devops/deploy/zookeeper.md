# Zookeeper 3.7.0

官方网站：https://zookeeper.apache.org/releases.html

下载地址：https://archive.apache.org/dist/zookeeper/

## 1. 安装

### 1.1 单机

#### 1.1.1 上传解压

```bash
cd /opt/software
tar -xzf apache-zookeeper-3.7.0-bin.tar.gz -C /opt/
mv /opt/apache-zookeeper-3.7.0-bin /opt/apache-zookeeper-3.7.0
sudo chown -R hadoop:hadoop /opt/apache-zookeeper-3.7.0 # 非root启动
```

#### 1.1.2 设置环境变量

```bash
vi /etc/profile
export ZK_HOME=/opt/apache-zookeeper-3.7.0
export PATH=$PATH:$ZK_HOME/bin
source /etc/profile
```

#### 1.1.3 修改配置

```bash
mkdir -p /opt/apache-zookeeper-3.7.0/data /opt/apache-zookeeper-3.7.0/logs
cd /opt/apache-zookeeper-3.7.0/conf
cp zoo_sample.cfg zoo.cfg
vi zoo.cfg
```

```conf
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/apache-zookeeper-3.7.0/data
dataLogDir=/opt/apache-zookeeper-3.7.0/logs
clientPort=2181
```

#### 1.1.4 启动服务

```bash
zkServer.sh start
jps
```

### 1.2 集群

#### 1.2.1 基础环境准备

将192.168.20.201单机复制三台，每台机器设置hostname

```bash
hostnamectl set-hostname node01 # 192.168.20.201 
hostnamectl set-hostname node02 # 192.168.20.202
hostnamectl set-hostname node03 # 192.168.20.203 
```

三台机器配置host

```bash
vi /etc/hosts
192.168.20.201 node01
192.168.20.202 node02
192.168.20.203 node03
```

#### 1.2.2 设置环境变量

```bash
vi /etc/profile
export ZK_HOME=/opt/apache-zookeeper-3.7.0
export PATH=$PATH:$ZK_HOME/bin
source /etc/profile
```

#### 1.2.3 环境变量分发

```bash
scp /etc/profile root@node02:/etc/
scp /etc/profile root@node03:/etc/
```

#### 1.2.4 修改配置

```bash
vi /opt/apache-zookeeper-3.7.0/conf/zoo.cfg
```

```conf
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/apache-zookeeper-3.7.0/data
dataLogDir=/opt/apache-zookeeper-3.7.0/logs
clientPort=2181
server.1=node01:2888:3888
server.2=node02:2888:3888
server.3=node03:2888:3888
```

#### 1.2.5 分发安装包

```bash
cd /opt/apache-zookeeper-3.7.0
scp -r /opt/apache-zookeeper-3.7.0 node02:$PWD
scp -r /opt/apache-zookeeper-3.7.0 node03:$PWD
```

#### 1.2.6 修改配置

每台机器设置各自myid

```bash 
touch /opt/apache-zookeeper-3.7.0/data/myid & echo 1 > /opt/apache-zookeeper-3.7.0/data/myid    # 192.168.20.201 
touch /opt/apache-zookeeper-3.7.0/data/myid & echo 2 > /opt/apache-zookeeper-3.7.0/data/myid    # 192.168.20.202 
touch /opt/apache-zookeeper-3.7.0/data/myid & echo 3 > /opt/apache-zookeeper-3.7.0/data/myid    # 192.168.20.203 
```

#### 1.2.7 启动服务

```shell
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh start   # 每个节点分别启动
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh status  # 检测节点状态
```

#### 1.2.8 客户端测试

PrettyZoo

下载地址：https://github.com/vran-dev/PrettyZoo/releases


## 2. 命令

```bash
# 服务端命令
zkServer.sh start   # 启动命令
zkServer.sh stop    # 停止命令
zkServer.sh restart # 重启命令
zkServer.sh status  # 查看集群节点状态

# 客户端命令
zkCli.sh            # 启动客户端
ls /                # 查看节点
get /test           # 查看节点数据
ls2 /test           # 查看该节点的子节点信息和属性信息
create /app1 hello              # 创建普通节点
create -s /app3 world           # 创建顺序节点
create -e /tempnode world       # 创建临时节点
create -s -e /tempnode2 aaa     # 创建顺序的临时节点
set /app1  xxx                  # 修改节点数据
delete /test                    # 删除的节点不能有子节点
rmr    /app1                    # 递归删除
```

## 3. 配置文件

```bash
# 快照文件snapshot的目录
dataDir=/opt/apache-zookeeper-3.7.0/data
# 事务日志的目录
dataLogDir=/opt/apache-zookeeper-3.7.0/logs
# 可以开启自动清理机制,自动清理tx log日志 频率是小时
autopurge.purgeInterval=48
# 需要保留的文件数目。默认是保留3个
autopurge.snapRetainCount=3 

# 客户端连接Zookeeper服务器的端口
clientPort=2181
# 客户端的并发连接数限制，默认值是60，将它设置为0表示取消对并发连接的限制
maxClientCnxns=60

# 服务器之间或客户端与服务器之间维持心跳的时间间隔，每个tickTime就会发送一个心跳。一个标准时间单元。所有时间都是以这个时间单元为基础，进行整数倍配置的。例如，session的最小超时时间是2*tickTime。
tickTime=2000
# LF初始通信时限
initLimit=10
# LF同步通信时限
syncLimit=5

server.1=node01:2888:3888
server.2=node02:2888:3888

## Metrics Providers
# https://prometheus.io Metrics Exporter
#metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
#metricsProvider.httpPort=7000
#metricsProvider.exportJvmInfo=true

```
