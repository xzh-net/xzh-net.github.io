# Zookeeper 3.7.0

ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby一个开源的实现，是Hadoop和Hbase的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等

- 官方网站：https://zookeeper.apache.org/releases.html
- 下载地址：https://archive.apache.org/dist/zookeeper/

## 1. 安装

### 1.1 单机

#### 1.1.1 下载解压

```bash
cd /opt/software
tar -zxf apache-zookeeper-3.7.0-bin.tar.gz -C /usr/local/
mv /usr/local/apache-zookeeper-3.7.0-bin /usr/local/zookeeper
```

#### 1.1.2 安装 Java

解压安装包

```bash
tar -zxvf jdk-8u202-linux-x64.tar.gz -C /usr/local/
```

配置环境变量

```bash
vim /etc/profile
```

添加：

```conf
export JAVA_HOME=/usr/local/jdk1.8.0_202
export PATH=$PATH:$JAVA_HOME/bin
```

使配置生效

```bash
source /etc/profile
```

#### 1.1.3 创建数据目录

```bash
mkdir -p /data/zookeeper/{data,logs}
```

#### 1.1.4 配置 zoo.cfg

```bash
cp /usr/local/zookeeper/conf/zoo_sample.cfg /usr/local/zookeeper/conf/zoo.cfg
vi /usr/local/zookeeper/conf/zoo.cfg
```

修改内容

```conf
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
clientPort=2181
```

#### 1.1.5 配置环境变量（可选）

```bash
vim /etc/profile
```

```conf
export ZK_HOME=/usr/local/zookeeper
export PATH=$PATH:$ZK_HOME/bin
```

```bash
source /etc/profile
```

#### 1.1.6 启动服务

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

#### 1.1.7 验证

连接服务端

```bash
/usr/local/zookeeper/bin/zkCli.sh -server localhost:2181
```

常用命令

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

#### 1.2.1 服务器准备

以3节点示例，所有节点配置 Host

```bash
vi /etc/hosts
192.168.20.201 node01
192.168.20.202 node02
192.168.20.203 node03
```

#### 1.2.2 创建数据目录

所有节点创建数据目录

```bash
mkdir -p /data/zookeeper/{data,logs}
```


#### 1.2.3 配置 zoo.cfg

所有节点修改配置

```bash
vi /usr/local/zookeeper/conf/zoo.cfg
```

```conf
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
clientPort=2181

# 集群配置 - 所有节点配置相同
server.1=node01:2888:3888
server.2=node02:2888:3888
server.3=node03:2888:3888
```

端口说明：
- 2888：节点间数据同步端口
- 3888：选举通信端口

#### 1.2.4 创建 myid 文件

```bash
# 在 node1 执行
echo 1 > /data/zookeeper/data/myid    # 192.168.20.201 
# 在 node2 执行
echo 2 > /data/zookeeper/data/myid    # 192.168.20.202 
# 在 node3 执行
echo 3 > /data/zookeeper/data/myid    # 192.168.20.203 
```

#### 1.2.5 配置环境变量（可选）

```bash
vim /etc/profile
```

```conf
export ZK_HOME=/usr/local/zookeeper
export PATH=$PATH:$ZK_HOME/bin
```

```bash
source /etc/profile
```

#### 1.2.6 启动集群

所有节点执行

```shell
/usr/local/zookeeper/bin/zkServer.sh start   # 启动
/usr/local/zookeeper/bin/zkServer.sh status  # 检测节点状态
```

#### 1.2.7 验证集群

```bash
/usr/local/zookeeper/bin/zkCli.sh -server localhost:2181
```

#### 1.2.8 客户端工具

下载地址：https://github.com/vran-dev/PrettyZoo/releases
