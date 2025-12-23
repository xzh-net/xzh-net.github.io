# Kafka 3.1.0

Apache Kafka 是一个开源分布式事件流平台

- 官方网站：https://kafka.apache.org

## 1. 安装

### 1.1 单机

#### 1.1.1 下载解压

```bash
cd /opt/software
tar -zxf kafka_2.13-3.1.0.tgz -C /usr/local
mv /usr/local/kafka_2.13-3.1.0 /usr/local/kafka
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
mkdir -p /data/kafka/logs
```

#### 1.1.4 配置 server.properties

修改配置文件
```bash
vi /usr/local/kafka/config/server.properties
```

```conf
broker.id=0
listeners=PLAINTEXT://:9092
log.dirs=/data/kafka/logs
zookeeper.connect=localhost:2181
```


#### 1.1.5 配置环境变量（可选）

```bash
vim /etc/profile
```

```conf
export KAFKA_HOME=/usr/local/kafka
export PATH=:$PATH:${KAFKA_HOME}
```

```bash
source /etc/profile
```

#### 1.1.6 启动服务

```bash
# 启动 zookeeper 自带单机版
nohup /usr/local/kafka/bin/zookeeper-server-start.sh /usr/local/kafka/config/zookeeper.properties &
# 启动 kafka
nohup /usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties &

# 关闭服务
/usr/local/kafka/bin/zookeeper-server-stop.sh
/usr/local/kafka/bin/kafka-server-stop.sh
```

#### 1.1.7 验证

```bash
# 创建主题
/usr/local/kafka/bin/kafka-topics.sh --create --topic product --partitions 1 --replication-factor 3 --bootstrap-server 127.0.0.1:9092
# 发送消息
/usr/local/kafka/bin/kafka-console-producer.sh --topic product --bootstrap-server 127.0.0.1:9092
# 消费
/usr/local/kafka/bin/kafka-console-consumer.sh --topic product --from-beginning --bootstrap-server 127.0.0.1:9092
# 删除主题
/usr/local/kafka/bin/kafka-topics.sh --delete --topic product --bootstrap-server 127.0.0.1:9092
```

### 1.2 Zookeeper 集群

#### 1.2.1 服务器准备

以3节点示例，所有节点配置 Host，不使用 kafka 自带的 Zookeeper，使用独立搭建的 Zookeeper 集群

```bash
vi /etc/hosts
192.168.20.201 node01
192.168.20.202 node02
192.168.20.203 node03
```

#### 1.2.2 创建数据目录

所有节点创建数据目录

```bash
mkdir -p /data/kafka/logs
```

#### 1.2.3 配置 server.properties

所有节点修改配置

```bash
vi /usr/local/kafka/config/server.properties
```

node01 节点

```conf
broker.id=1
listeners=PLAINTEXT://:9092
log.dirs=/data/kafka/logs
zookeeper.connect=node01:2181,node02:2181,node03:2181/kafka
```


node02 节点

```conf
broker.id=2
listeners=PLAINTEXT://:9092
log.dirs=/data/kafka/logs
zookeeper.connect=node01:2181,node02:2181,node03:2181/kafka
```


node03 节点

```conf
broker.id=3
listeners=PLAINTEXT://:9092
log.dirs=/data/kafka/logs
zookeeper.connect=node01:2181,node02:2181,node03:2181/kafka
```


#### 1.2.4 配置环境变量（可选）

```bash
vim /etc/profile
```

```conf
export KAFKA_HOME=/usr/local/kafka
export PATH=:$PATH:${KAFKA_HOME}
```

```bash
source /etc/profile
```

#### 1.2.5 启动集群

所有节点执行

```bash
# 启动 zookeeper 独立集群
/usr/local/zookeeper/bin/zkServer.sh start
# 启动 kafka
nohup /usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties &
```

#### 1.2.6 验证集群

```bash
# node01 节点执行创建主题
/usr/local/kafka/bin/kafka-topics.sh --create --topic product --partitions 6 --replication-factor 3 --bootstrap-server 127.0.0.1:9092
# 其他节点查询主题，有数据表示消息同步成功
/usr/local/kafka/bin/kafka-topics.sh --bootstrap-server node01:9092 --list
```

### 1.3 KRaft 集群

KRaft 是 Kafka 自 2.8 版本开始引入的新的元数据管理机制，用于替代 ZooKeeper。以下是一个三节点 KRaft 集群的搭建步骤

#### 1.3.1 生成集群ID

在任意节点执行以下命令生成集群ID

```bash
/usr/local/kafka/bin/kafka-storage.sh random-uuid
```

例如： `Rj3jxXL5SSepoaWBf0LYIA`，后面配置会用到。

#### 1.3.2 配置 kraft/server.properties

所有节点修改配置

```bash
vi /usr/local/kafka/config/kraft/server.properties
```

node01 节点

```conf
# 节点基础配置
node.id=1
process.roles=controller,broker

# 网络配置
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://node01:9092
controller.listener.names=CONTROLLER

# Controller配置
controller.quorum.voters=1@node01:9093,2@node02:9093,3@node03:9093

# 存储配置
log.dirs=/data/kafka/logs

# 集群配置
cluster.id=LAY6pXKJQlCj8G8-PSFXjA
```


node02 节点

```conf
# 节点基础配置
node.id=2
process.roles=controller,broker

# 网络配置
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://node02:9092
controller.listener.names=CONTROLLER

# Controller配置
controller.quorum.voters=1@node01:9093,2@node02:9093,3@node03:9093

# 存储配置
log.dirs=/data/kafka/logs

# 集群配置
cluster.id=LAY6pXKJQlCj8G8-PSFXjA
```


node03 节点

```conf
# 节点基础配置
node.id=3
process.roles=controller,broker

# 网络配置
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://node03:9092
controller.listener.names=CONTROLLER

# Controller配置
controller.quorum.voters=1@node01:9093,2@node02:9093,3@node03:9093

# 存储配置
log.dirs=/data/kafka/logs

# 集群配置
cluster.id=LAY6pXKJQlCj8G8-PSFXjA
```

#### 1.3.3 格式化存储


用集群ID格式化三台机器的kafka存储目录

```bash
/usr/local/kafka/bin/kafka-storage.sh format -t Rj3jxXL5SSepoaWBf0LYIA -c /usr/local/kafka/config/kraft/server.properties
```

格式化结果查看

```bash
ls -la /data/kafka/logs
```

#### 1.3.4 启动集群

三台机器分别执行
```bash
${KAFKA_HOME}/bin/kafka-server-start.sh -daemon ${KAFKA_HOME}/config/kraft/server.properties
```

#### 1.3.5 验证集群

```bash
# node01 节点执行创建主题
/usr/local/kafka/bin/kafka-topics.sh --create --topic product --partitions 6 --replication-factor 3 --bootstrap-server 127.0.0.1:9092
# 其他节点查询主题，有数据表示消息同步成功
/usr/local/kafka/bin/kafka-topics.sh --bootstrap-server node01:9092 --list
```

## 2. 可视化监控

下载地址：http://download.kafka-eagle.org/

### 2.1 上传解压

```bash
cd /opt/software
tar -zxvf kafka-eagle-bin-3.0.1.tar.gz
cd /opt/kafka-eagle-bin-3.0.1 && tar -zxvf efak-web-3.0.1-bin.tar.gz -C /usr/local
mv /usr/local/efak-web-3.0.1 /usr/local/efak-web
```

### 2.2 配置环境变量

```bash
vi /etc/profile
```

```conf
export KE_HOME=/usr/local/efak-web
export PATH=$PATH:$KE_HOME/bin
```

```bash
source /etc/profile
```

### 2.3 修改配置

```bash
vi /usr/local/efak-web/conf/system-config.properties
```

```conf
efak.zk.cluster.alias=cluster1
cluster1.zk.list=node01:2181,node02:2181,node03:2181

efak.driver=com.mysql.cj.jdbc.Driver
efak.url=jdbc:mysql://127.0.0.1:3306/ke?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
efak.username=root
efak.password=root
```

### 2.4 启动服务

```bash
cd ${KE_HOME}/bin
chmod +x ke.sh 
# 启动
ke.sh start
# 停止
ke.sh stop
```

访问地址：http://192.168.3.201:8048 admin/123456


### 2.5 常用脚本

#### 2.5.1 xcall

```bash
#!/bin/bash

for host in node01 node02 node03
do
    echo =============== $host ===============
    ssh $host /usr/local/jdk1.8.0_202/bin/jps 
done

```

#### 2.5.2 zookeeper.sh

```bash
#!/bin/bash

case $1 in
"start"){
	for i in node01 node02 node03
	do
		echo  ------------- zookeeper $i 启动 ------------
		ssh $i "/usr/local/zookeeper/bin/zkServer.sh start"
	done
}
;;
"stop"){
	for i in node01 node02 node03
	do
		echo  ------------- zookeeper $i 停止 ------------
		ssh $i "/usr/local/zookeeper/bin/zkServer.sh stop"
	done
}
;;
"status"){
	for i in node01 node02 node03
	do
		echo  ------------- zookeeper $i 状态 ------------
		ssh $i "/usr/local/zookeeper/bin/zkServer.sh status"
	done
}
;;
esac
```

```bash
vi /etc/profile.d/my_env.sh
# 添加
export JAVA_HOME=/usr/local/jdk1.8.0_202
export PATH=$PATH:$JAVA_HOME/bin
```


#### 2.5.3 kafka.sh

```bash
#!/bin/bash

case $1 in
"start"){
	for i in node01 node02 node03
	do
		echo  ------------- 启动 $i kafka ------------
		ssh $i "source /etc/profile;export JMX_PORT=9999;/usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties"
	done
}
;;
"stop"){
	for i in node01 node02 node03
	do
		echo  ------------- 停止 $i kafka ------------
		ssh $i "/usr/local/kafka/bin/kafka-server-stop.sh"
	done
}
;;
"kill"){
	for i in node01 node02 node03
	do
		echo  ------------- 停止 $i kafka ------------
		ssh $i "source /etc/profile;jps |grep Kafka |cut -d' ' -f1 |xargs kill -s 9"
	done
}
;;
esac
```

#### 2.5.4 kraft.sh

```bash
#!/bin/bash

case $1 in
"start"){
	for i in node01 node02 node03
	do
		echo  ------------- 启动 $i kafka ------------
		ssh $i "source /etc/profile;export JMX_PORT=9999;/usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/kraft/server.properties"
	done
}
;;
"stop"){
	for i in node01 node02 node03
	do
		echo  ------------- 停止 $i kafka ------------
		ssh $i "/usr/local/kafka/bin/kafka-server-stop.sh"
	done
}
;;
"kill"){
	for i in node01 node02 node03
	do
		echo  ------------- 停止 $i kafka ------------
		ssh $i "source /etc/profile;jps |grep Kafka |cut -d' ' -f1 |xargs kill -s 9"
	done
}
;;
esac
```
