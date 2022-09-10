# Pulsar 2.10.1

官方网站：https://pulsar.apache.org

下载地址：https://pulsar.apache.org/download

## 1. 安装

### 1.1 单机

#### 1.1.1 上传解压

```bash
cd /opt/software
tar -zxvf apache-pulsar-2.10.1-bin.tar.gz -C /opt
```

#### 1.1.2 启动

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar standalone
```

#### 1.1.3 客户端测试

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-client consume my-topic -s "first-subscription"    # 监听数据
./pulsar-client produce my-topic --messages "hello-pulsar"  # 模拟生产数据
```


### 1.2 集群

#### 1.2.1 集群规划

- node01 ZooKeeper bookie broker
- node02 ZooKeeper bookie broker
- node03 ZooKeeper bookie broker

#### 1.2.2 上传解压

```bash
cd /opt/software
tar -zxvf apache-pulsar-2.10.1-bin.tar.gz -C /opt
```

#### 1.2.3 修改bookkeeper集群配置文件

```bash
cd /opt/apache-pulsar-2.10.1/conf
vim bookkeeper.conf
```

编辑内容

```conf
advertisedAddress=node01.xuzhihao.net
journalDirectory=/opt/apache-pulsar-2.10.1/data/bookkeeper/journal
ledgerDirectories=/opt/apache-pulsar-2.10.1/data/bookkeeper/ledgers
zkServers=node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181
```

#### 1.2.4 修改broker集群的配置文件

```bash
cd /opt/apache-pulsar-2.10.1/conf
vim broker.conf
```

编辑内容

```conf
clusterName=pulsar-cluster
zookeeperServers=node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181
configurationStoreServers=node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181
advertisedAddress=node01.xuzhihao.net
```

#### 1.2.5 分发安装包

```
cd /opt/apache-pulsar-2.10.1
scp -r /opt/apache-pulsar-2.10.1 node02:$PWD
scp -r /opt/apache-pulsar-2.10.1 node03:$PWD
```

#### 1.2.6 修改分发后的broker的地址和bookies地址

1. node02节点执行

```bash
cd /opt/apache-pulsar-2.10.1/conf
vim bookkeeper.conf
# 编辑内容
advertisedAddress=node02.xuzhihao.net
```

```bash
vim broker.conf
# 编辑内容
advertisedAddress=node02.xuzhihao.net
```

2. node03节点执行

```bash
cd /opt/apache-pulsar-2.10.1/conf
vim bookkeeper.conf
# 编辑内容
advertisedAddress=node03.xuzhihao.net
```

```bash
vim broker.conf
# 编辑内容
advertisedAddress=node03.xuzhihao.net
```

#### 1.2.7 启动zk集群

```bash
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh start   # 每个节点分别启动
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh status  # 检测节点状态
```

#### 1.2.8 初始化元数据

```bash
cd /opt/apache-pulsar-2.10.1/bin

./pulsar initialize-cluster-metadata \
--cluster pulsar-cluster \
--zookeeper node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181 \
--configuration-store node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181 \
--web-service-url http://node01.xuzhihao.net:8080,node02.xuzhihao.net:8080,node03.xuzhihao.net:8080 \
--web-service-url-tls https://node01.xuzhihao.net:8443,node02.xuzhihao.net:8443,node03.xuzhihao.net:8443 \
--broker-service-url pulsar://node01.xuzhihao.net:6650,node02.xuzhihao.net:6650,node03.xuzhihao.net:6650 \
--broker-service-url-tls pulsar+ssl://node01.xuzhihao.net::6651,node02.xuzhihao.net:6651,node03.xuzhihao.net:6651

./bookkeeper shell metaformat   # 初始化bookkeeper集群: 若出现提示输入Y/N: 请输入Y
```

#### 1.2.9 集群模式启动

1. 启动bookkeeper

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-daemon start bookie    # 三个节点都需要依次启动
./bookkeeper shell bookiesanity # 验证是否启动
```

2. 启动Broker

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-daemon start broker    # 三个节点都需要依次启动
./pulsar-admin brokers list pulsar-cluster  # 验证是否启动
```

#### 1.2.10 集群模式测试

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-client consume persistent://public/default/test -s "consumer-test"         # 监听数据
./pulsar-client produce persistent://public/default/test --messages "hello-pulsar"  # 模拟生产数据
```

## 2. 可视化监控