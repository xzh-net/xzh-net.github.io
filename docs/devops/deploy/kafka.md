# Kafka 2.13-3.1.0

官方网站：https://kafka.apache.org

## 1. 安装

### 1.1 单机

1. 上传解压

```bash
cd /opt/software
tar -xzf kafka_2.13-3.1.0.tgz -C /opt/
```

2. 设置环境变量

```bash
vim /etc/profile
export KAFKA_HOME=/opt/kafka_2.13-3.1.0
export PATH=:$PATH:${KAFKA_HOME}
source /etc/profile
```

3. 修改配置

```bash
cd /opt/kafka_2.13-3.1.0/
mkdir -p /opt/kafka_2.13-3.1.0/data
vi config/server.properties

broker.id=0
listeners=PLAINTEXT://192.168.3.200:9092     # brokder对外提供的服务入口地址
zookeeper.connect=192.168.3.200:2181
log.dirs=/opt/kafka_2.13-3.1.0/data
```

4. 启动服务器

```bash
cd /opt/kafka_2.13-3.1.0/
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &   # 自2.8.0取消了zookeeper
nohup bin/kafka-server-start.sh config/server.properties &
```

5. 测试

```bash
bin/kafka-topics.sh --create --topic product --partitions 2 --replication-factor 3 --bootstrap-server 192.168.3.200:9092 	# 创建主题
bin/kafka-console-producer.sh --topic product --bootstrap-server 192.168.3.200:9092                  # 发送消息
bin/kafka-console-consumer.sh --topic product --from-beginning --bootstrap-server 192.168.3.200:9092 # 消费
```

### 1.2 集群

1. 将Kafka的安装包上传到虚拟机，并解压

```bash
cd /opt/software
tar -xzf kafka_2.13-3.1.0.tgz -C /opt/
```

2. 修改server.properties

```bash
cd /opt/kafka_2.13-3.1.0/
vi config/server.properties

broker.id=1
log.dirs=/opt/kafka_2.13-3.1.0/data
zookeeper.connect=node01:2181,node02:2181,node03:2181/kafka
```

3. 将安装好的kafka复制到另外两台服务器

```bash
scp -r /opt/kafka_2.13-3.1.0/ node02:/opt/kafka_2.13-3.1.0/
scp -r /opt/kafka_2.13-3.1.0/ node03:/opt/kafka_2.13-3.1.0/

# 修改另外两个节点的broker.id分别为2和3
# 192.168.3.202
cd /opt/kafka_2.13-3.1.0/
vi config/server.properties
broker.id=2

# 192.168.3.203
cd /opt/kafka_2.13-3.1.0/
vi config/server.properties
broker.id=3
```

4. 配置环境变量

```bash
vim /etc/profile
export KAFKA_HOME=/opt/kafka_2.13-3.1.0
export PATH=:$PATH:${KAFKA_HOME}
source /etc/profile

# 分发到各个节点
scp /etc/profile node02:/etc/profile
scp /etc/profile node03:/etc/profile
# 每个节点加载环境变量
source /etc/profile
```

5. 启动服务器

```bash
cd /opt/kafka_2.13-3.1.0/
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &   # 启动ZooKeeper
nohup bin/kafka-server-start.sh config/server.properties &          # 启动Kafka
# 测试Kafka集群是否启动成功
bin/kafka-topics.sh --bootstrap-server node01:9092 --list
```

### 1.3 kraft

1. 修改配置

```bash
cd /opt/kafka_2.13-3.1.0/config/kraft
mkdir -p /opt/kafka_2.13-3.1.0/config/kraft/logs
server.properties

# 修改以下配置
process.roles=broker,controller	# 数据节点，控制节点
node.id=1	# 对应broker.id
controller.quorum.voters=1@node01:9093,2@node02:9093,3@node03:9093 # 对应zk
advertised.listeners=PLAINTEXT://node01:9092		# 对外暴漏端口，对应主机名称
log.dirs=/opt/kafka_2.13-3.1.0/config/kraft/logs
```

2. 内容分发

修改node02和node02的节点id和暴漏端口

```bash
scp -r /opt/kafka_2.13-3.1.0/config/kraft/ node02:/opt/kafka_2.13-3.1.0/config/
scp -r /opt/kafka_2.13-3.1.0/config/kraft/ node03:/opt/kafka_2.13-3.1.0/config/
```

3. 初始化

```bash
cd $KAFKA_HOME
bin/kafka-storage.sh random-uuid	# 生成id
```

用该id格式化三台机器的kafka存储目录

```bash
cd $KAFKA_HOME
bin/kafka-storage.sh format -t 0T25c_coRNqGuGvssegx3Q -c /opt/kafka_2.13-3.1.0/config/kraft/server.properties
```

4. 启动服务

三台机器分别执行
```bash
${KAFKA_HOME}/bin/kafka-server-start.sh -daemon ${KAFKA_HOME}/config/kraft/server.properties
```

## 2. 脚本

### 2.1 xcall

```bash
#!/bin/bash

for host in node01 node02 node03
do
    echo =============== $host ===============
    ssh $host /usr/local/jdk1.8.0_211/bin/jps 
done

```

### 2.2 zk.sh

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


### 2.3 ka.sh

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

### 2.4 kraft.sh

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

## 3. 监控

下载地址：http://download.kafka-eagle.org/

1. 上传解压

```bash
cd /opt/software
tar -zxvf kafka-eagle-bin-2.1.0.tar.gz -C /opt/
cd /opt/kafka-eagle-bin-2.1.0 && tar -zxvf efak-web-2.1.0-bin.tar.gz -C /opt/
rm -rf /opt/kafka-eagle-bin-2.1.0
cd /opt/efak-web-2.1.0
```

2. 设置环境变量

```bash
vi /etc/profile
export KE_HOME=/opt/efak-web-2.1.0
export PATH=$PATH:$KE_HOME/bin
source /etc/profile
```

3. 修改配置

```bash
cd /opt/efak-web-2.1.0/conf

vi system-config.properties
efak.zk.cluster.alias=cluster1
cluster1.zk.list=node01:2181,node02:2181,node03:2181

efak.driver=com.mysql.cj.jdbc.Driver
efak.url=jdbc:mysql://127.0.0.1:3306/ke?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
efak.username=root
efak.password=123456
```

vi kafka-server-start.sh 
```bash
export KAFKA_HEAP_OPTS="-server -Xms2G -Xmx2G -XX:PermSize=128m -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:ParallelGCThreads=8 -XX:ConcGCThreads=5 -XX:InitiatingHeapOccupancyPercent=70"
export JMX_PORT="9999"
```

4. 启动服务

```bash
cd ${KE_HOME}/bin
chmod +x ke.sh 
ke.sh start
ke.sh stop
ke.sh status
ke.sh stats
ke.sh restart
```

访问地址：http://192.168.3.200:8048 admin/123456
