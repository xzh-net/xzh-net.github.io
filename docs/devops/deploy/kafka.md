# Kafka 2.13-3.1.0

## 1. 安装

1. 下载解压

- https://kafka.apache.org/

```bash
tar -xzf kafka_2.13-3.1.0.tgz -C /opt/
cd /opt/kafka_2.13-3.1.0/
```

2. 配置

```bash
vi config/server.properties
broker.id=0
listeners=PLAINTEXT://172.17.17.200:9092     # brokder对外提供的服务入口地址
zookeeper.connect=172.17.17.200:2181
```

3. 启动

```bash
# 新版内置了zookeeper
bin/zookeeper-server-start.sh config/zookeeper.properties
bin/kafka-server-start.sh config/server.properties
```

4. 测试

```bash
bin/kafka-topics.sh --create --topic topic1 --bootstrap-server 172.17.17.200:9092                   # 创建主题
bin/kafka-console-producer.sh --topic topic1 --bootstrap-server 172.17.17.200:9092                  # 发送消息
bin/kafka-console-consumer.sh --topic topic1 --from-beginning --bootstrap-server 172.17.17.200:9092 # 消费
```

## 2. 监控

1. 下载解压

- http://download.kafka-eagle.org/

```bash
tar -zxvf kafka-eagle-bin-2.1.0.tar.gz -C /opt/
cd /opt/kafka-eagle-bin-2.1.0
tar -zxvf efak-web-2.1.0-bin.tar.gz
cd /opt/kafka-eagle-bin-2.1.0/efak-web-2.1.0
```

2. 系统配置
```bash
vi /etc/profile
export KE_HOME=/opt/kafka-eagle-bin-2.1.0/efak-web-2.1.0
export PATH=$PATH:$KE_HOME/bin
source /etc/profile
```

3. 应用配置
```bash
cd ${KE_HOME}/conf
vi system-config.properties
efak.zk.cluster.alias=cluster1
cluster1.zk.list=172.17.17.200:2181
```

4. 启动
```bash
cd ${KE_HOME}/bin
chmod +x ke.sh 
ke.sh start
ke.sh restart
ke.sh stop
```


## 3. 集群

1. 解压

```bash
tar -xzf kafka_2.13-3.1.0.tgz -C /opt/
cd /opt/kafka_2.13-3.1.0/

# 修改 server.properties
vim config/server.properties
# 指定broker的id
broker.id=0
# 指定Kafka数据的位置
log.dirs=/opt/kafka_2.13-3.1.0/data
# 配置zk的三个节点
zookeeper.connect=172.17.17.200:2181,172.17.17.201:2181,172.17.17.202:2181
```

2. 复制

```bash
cd /opt
scp -r kafka_2.13-3.1.0/ 172.17.17.201:$PWD
scp -r kafka_2.13-3.1.0/ 172.17.17.202:$PWD

# 修改另外两个节点的broker.id分别为1和2
#---------172.17.17.201--------------
cd /opt/kafka_2.13-3.1.0/
vim config/server.properties
broker.id=1

#---------172.17.17.202--------------
cd /opt/kafka_2.13-3.1.0/
vim config/server.properties
broker.id=2
```

配置KAFKA_HOME环境变量
```bash
vim /etc/profile
export KAFKA_HOME=/opt/kafka_2.13-3.1.0/
export PATH=:$PATH:${KAFKA_HOME}

#分发到各个节点
scp /etc/profile 172.17.17.201:$PWD
scp /etc/profile 172.17.17.202:$PWD
每个节点加载环境变量
source /etc/profile
```


3. 启动

```bash
cd /opt/kafka_2.13-3.1.0/
# 依次启动ZooKeeper
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &
# 依次启动Kafka
nohup bin/kafka-server-start.sh config/server.properties &
```

4. 测试
```bash
# 测试Kafka集群是否启动成功
bin/kafka-topics.sh --bootstrap-server 172.17.17.201:9092 --list
```