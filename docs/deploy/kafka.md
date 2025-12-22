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
zookeeper.connect=192.168.3.200:2181
log.dirs=/data/kafka/logs
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
# 启动 zookeeper
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

### 1.2 集群

#### 1.2.1 服务器准备

以3节点示例，所有节点配置Host，不使用 kafka 自带的 zookeeper，使用独立搭建的 zookeeper 集群

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
zookeeper.connect=node01:2181,node02:2181,node03:2181/kafka
log.dirs=/data/kafka/logs
```


node02 节点

```conf
broker.id=2
listeners=PLAINTEXT://:9092
zookeeper.connect=node01:2181,node02:2181,node03:2181/kafka
log.dirs=/data/kafka
```


node03 节点

```conf
broker.id=3
listeners=PLAINTEXT://:9092
zookeeper.connect=node01:2181,node02:2181,node03:2181/kafka
log.dirs=/data/kafka
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
# 启动 zookeeper
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

### 1.3 kraft

#### 1.3.1 修改配置

```bash
cd /opt/kafka_2.13-3.1.0/config/kraft
mkdir -p /opt/kafka_2.13-3.1.0/data
```

```bash
vi server.properties
```

修改配置
```conf
process.roles=broker,controller	# 数据节点，控制节点
node.id=1	# 对应broker.id
controller.quorum.voters=1@node01:9093,2@node02:9093,3@node03:9093 # 对应zk
advertised.listeners=PLAINTEXT://node01:9092		# 对外暴漏端口，对应主机名称
log.dirs=/opt/kafka_2.13-3.1.0/data
```

#### 1.3.2 内容分发

修改node02和node02的节点id和暴漏端口

```bash
scp -r /opt/kafka_2.13-3.1.0/config/kraft/ node02:/opt/kafka_2.13-3.1.0/config/
scp -r /opt/kafka_2.13-3.1.0/config/kraft/ node03:/opt/kafka_2.13-3.1.0/config/
```

#### 1.3.3 初始化

```bash
cd $KAFKA_HOME
bin/kafka-storage.sh random-uuid	# 生成id
```

用该id格式化三台机器的kafka存储目录

```bash
cd $KAFKA_HOME
bin/kafka-storage.sh format -t 0T25c_coRNqGuGvssegx3Q -c /opt/kafka_2.13-3.1.0/config/kraft/server.properties
```

#### 1.3.4 启动服务

三台机器分别执行
```bash
${KAFKA_HOME}/bin/kafka-server-start.sh -daemon ${KAFKA_HOME}/config/kraft/server.properties
```


## 2. 可视化监控

下载地址：http://download.kafka-eagle.org/

### 2.1 上传解压

```bash
cd /opt/software
tar -zxvf kafka-eagle-bin-3.0.1.tar.gz -C /opt/
cd /opt/kafka-eagle-bin-3.0.1 && tar -zxvf efak-web-3.0.1-bin.tar.gz -C /opt/
rm -rf /opt/kafka-eagle-bin-3.0.1
cd /opt/efak-web-3.0.1
```

### 2.2 设置环境变量

```bash
vi /etc/profile
```

```conf
export KE_HOME=/opt/efak-web-3.0.1
export PATH=$PATH:$KE_HOME/bin
```

```bash
source /etc/profile
```

### 2.3 修改配置

```bash
cd /opt/efak-web-3.0.1/conf
```

```bash
vi system-config.properties
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
ke.sh start | stop | status | restart
```

访问地址：http://192.168.3.201:8048 admin/123456


## 3. 脚本

### 3.1 xcall

```bash
#!/bin/bash

for host in node01 node02 node03
do
    echo =============== $host ===============
    ssh $host /usr/local/jdk1.8.0_202/bin/jps 
done

```

### 3.2 zk.sh

```bash
#!/bin/bash

case $1 in
"start"){
	for i in node01 node02 node03
	do
		echo  ------------- zookeeper $i 启动 ------------
		ssh $i "/opt/apache-zookeeper-3.7.0/bin/zkServer.sh start"
	done
}
;;
"stop"){
	for i in node01 node02 node03
	do
		echo  ------------- zookeeper $i 停止 ------------
		ssh $i "/opt/apache-zookeeper-3.7.0/bin/zkServer.sh stop"
	done
}
;;
"status"){
	for i in node01 node02 node03
	do
		echo  ------------- zookeeper $i 状态 ------------
		ssh $i "/opt/apache-zookeeper-3.7.0/bin/zkServer.sh status"
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


### 3.3 ka.sh

```bash
#!/bin/bash

case $1 in
"start"){
	for i in node01 node02 node03
	do
		echo  ------------- 启动 $i kafka ------------
		ssh $i "source /etc/profile;export JMX_PORT=9999;${KAFKA_HOME}/bin/kafka-server-start.sh -daemon ${KAFKA_HOME}/config/server.properties"
	done
}
;;
"stop"){
	for i in node01 node02 node03
	do
		echo  ------------- 停止 $i kafka ------------
		ssh $i "/opt/kafka_2.13-3.1.0/bin/kafka-server-stop.sh"
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

### 3.4 kraft.sh

```bash
#!/bin/bash

case $1 in
"start"){
	for i in node01 node02 node03
	do
		echo  ------------- 启动 $i kafka ------------
		ssh $i "source /etc/profile;export JMX_PORT=9999;${KAFKA_HOME}/bin/kafka-server-start.sh -daemon ${KAFKA_HOME}/config/kraft/server.properties"
	done
}
;;
"stop"){
	for i in node01 node02 node03
	do
		echo  ------------- 停止 $i kafka ------------
		ssh $i "/opt/kafka_2.13-3.1.0/bin/kafka-server-stop.sh"
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
