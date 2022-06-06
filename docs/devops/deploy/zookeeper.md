# Zookeeper 3.7.0

- https://zookeeper.apache.org/releases.html
- https://downloads.apache.org/zookeeper/zookeeper-3.7.0/

## 1. 配置

```conf
# 快照文件snapshot的目录
dataDir=/data/zookeeper/data
# 事务日志的目录
dataLogDir=/data/zookeeper/datalogs
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

## 3. 集群

解压
```bash
cd ~
tar -xzf apache-zookeeper-3.7.0-bin.tar.gz -C /opt/
mv /opt/apache-zookeeper-3.7.0-bin /opt/apache-zookeeper-3.7.0
sudo chown -R hadoop:hadoop /opt/apache-zookeeper-3.7.0     # 非root启动
```

设置环境变量
```bash
vi /etc/profile
export ZK_HOME=/opt/apache-zookeeper-3.7.0
export PATH=$PATH:$ZK_HOME/bin
source /etc/profile
```

修改配置
```bash
cd /opt/apache-zookeeper-3.7.0/conf
cp zoo_sample.cfg zoo.cfg
vi zoo.cfg
```

配置文件
```conf
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/apache-zookeeper-3.7.0/data
dataLogDir=/opt/apache-zookeeper-3.7.0/logs
clientPort=2181
server.1=192.168.1.1:2888:3888
server.2=192.168.1.2:2888:3888
server.3=192.168.1.3:2888:3888
```

- 2888为组成zookeeper服务器之间的通信端口，3888为用来选举leader的端口
- 在data目录下新建一个myid文件，里面只包括该节点的id

```bash
echo 1 > /opt/apache-zookeeper-3.7.0/data/myid
```

将配置之后的zookeeper，分发到其他节点上，并修改myid，执行启动命令 zkServer.sh start
```bash
scp -r  /opt/apache-zookeeper-3.7.0 node02:/opt/
scp -r  /opt/apache-zookeeper-3.7.0 node03:/opt/
```

## 4. 可视化

PrettyZoo的安装包，下载地址：https://github.com/vran-dev/PrettyZoo/releases
