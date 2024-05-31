# Kafka 2.13-3.1.0

Apache Kafka 是一个开源分布式事件流平台

- 官方网站：https://kafka.apache.org

## 1. 安装

### 1.1 单机

#### 1.1.1 上传解压

```bash
cd /opt/software
tar -zxf kafka_2.13-3.1.0.tgz -C /opt/
```

#### 1.1.2 设置环境变量

```bash
vim /etc/profile
```

```conf
export KAFKA_HOME=/opt/kafka_2.13-3.1.0
export PATH=:$PATH:${KAFKA_HOME}
```

```bash
source /etc/profile
```

#### 1.1.3 修改配置

```bash
cd /opt/kafka_2.13-3.1.0/
mkdir -p /opt/kafka_2.13-3.1.0/data
```

```bash
vi config/server.properties
```

```conf
broker.id=0
# brokder对外提供的服务入口地址
listeners=PLAINTEXT://192.168.3.200:9092
zookeeper.connect=192.168.3.200:2181
log.dirs=/opt/kafka_2.13-3.1.0/data
```

#### 1.1.4 启动服务

```bash
cd /opt/kafka_2.13-3.1.0/
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &
nohup bin/kafka-server-start.sh config/server.properties &
```

#### 1.1.5 客户端测试

```bash
bin/kafka-topics.sh --create --topic product --partitions 2 --replication-factor 3 --bootstrap-server 192.168.3.200:9092 	# 创建主题
bin/kafka-console-producer.sh --topic product --bootstrap-server 192.168.3.200:9092                                         # 发送消息
bin/kafka-console-consumer.sh --topic product --from-beginning --bootstrap-server 192.168.3.200:9092                        # 消费
```

### 1.2 集群

#### 1.2.1 上传解压 

```bash
cd /opt/software
tar -zxf kafka_2.13-3.1.0.tgz -C /opt/
```

#### 1.2.2 修改配置

```bash
cd /opt/kafka_2.13-3.1.0/
```

```bash
vi config/server.properties
```

```conf
broker.id=1
log.dirs=/opt/kafka_2.13-3.1.0/data
zookeeper.connect=node01:2181,node02:2181,node03:2181/kafka
```

#### 1.2.3 内容分发

将安装好的kafka复制到另外两台服务器

```bash
scp -r /opt/kafka_2.13-3.1.0/ node02:/opt/kafka_2.13-3.1.0/
scp -r /opt/kafka_2.13-3.1.0/ node03:/opt/kafka_2.13-3.1.0/
```

修改192.168.3.202节点的broker.id
```bash
cd /opt/kafka_2.13-3.1.0/
vi config/server.properties
broker.id=2
```

修改192.168.3.203节点的broker.id
```bash
cd /opt/kafka_2.13-3.1.0/
vi config/server.properties
broker.id=3
```

#### 1.2.4 设置环境变量

```bash
vim /etc/profile
export KAFKA_HOME=/opt/kafka_2.13-3.1.0
export PATH=:$PATH:${KAFKA_HOME}
source /etc/profile
```

分发到各个节点

```bash
scp /etc/profile node02:/etc/profile
scp /etc/profile node03:/etc/profile
# 每个节点加载环境变量
source /etc/profile
```

#### 1.2.5 启动服务

```bash
cd /opt/kafka_2.13-3.1.0/
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &   # 启动ZooKeeper
nohup bin/kafka-server-start.sh config/server.properties &          # 启动Kafka
# 测试Kafka集群是否启动成功
bin/kafka-topics.sh --bootstrap-server node01:9092 --list
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
