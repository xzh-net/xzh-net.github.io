# Zookeeper 3.7.0

ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby一个开源的实现，是Hadoop和Hbase的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等

- 官方网站：https://zookeeper.apache.org/releases.html
- 下载地址：https://archive.apache.org/dist/zookeeper/

## 1. 安装

### 1.1 单机

#### 1.1.1 上传解压

```bash
cd /opt/software
tar -zxf apache-zookeeper-3.7.0-bin.tar.gz -C /usr/local/
mv /usr/local/apache-zookeeper-3.7.0-bin /usr/local/zookeeper
```

#### 1.1.2 设置环境变量

```bash
vim /etc/profile
```

```conf
export JAVA_HOME=/usr/local/jdk1.8.0_202
export PATH=$PATH:$JAVA_HOME/bin

export ZK_HOME=/usr/local/zookeeper
export PATH=$PATH:$ZK_HOME/bin
```

```bash
source /etc/profile
```

#### 1.1.3 修改配置

创建数据和日志目录

```bash
mkdir -p /data/zookeeper/{data,logs} -p
```

修改配置文件

```bash
cp /usr/local/zookeeper/conf/zoo_sample.cfg /usr/local/zookeeper/conf/zoo.cfg
vi /usr/local/zookeeper/conf/zoo.cfg
```

```conf
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
clientPort=2181
```

#### 1.1.4 启动服务

```bash
# 启动服务
/usr/local/zookeeper/bin/zkServer.sh start
# 停止服务
/usr/local/zookeeper/bin/zkServer.sh stop    
# 重启服务
/usr/local/zookeeper/bin/zkServer.sh restart
# 查看进程
jps
```

#### 1.1.5 验证

```bash
/usr/local/zookeeper/bin/zkCli.sh -server localhost:2181
```

```bash
ls /                # 查看所有节点
create /app test    # 创建持久节点
ls /app             # 查看该节点的子节点信息和属性信息
get /app            # 查看节点数据
set /app dev               # 修改节点数据
delete /app                # 删除的节点不能有子节点
deleteall /app             # 递归删除
create -s /snode sdata       # 创建顺序节点
create -e /enode edata       # 创建临时节点
create -s -e /senode sedata  # 创建顺序的临时节点
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
admin.serverPort=8089
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

#### 1.2.8 客户端工具

下载地址：https://github.com/vran-dev/PrettyZoo/releases


