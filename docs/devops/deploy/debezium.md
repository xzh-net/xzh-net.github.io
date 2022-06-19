# Debezium 1.7.2

官方网站：https://debezium.io/

## 1. MySQL

### 1.1 MySQL准备

1. 开启binlog

```bash
vi /etc/my.cnf

server_id=1
log_bin=mysql-bin
binlog_format=ROW
```

重启服务校验是否开启成功
```bash
systemctl restart mysqld 
mysql -uroot -p123456 -e "show variables like 'log_bin%'";
```

2. 准备测试库和表

```bash
mysql -uroot -p123456
CREATE DATABASE `test` CHARACTER SET utf8 COLLATE utf8_general_ci;
use test;
create table stu(id int primary key, name varchar(255), age int);
```

### 1.2 安装Mysql Connector

1. 上传解压

```bash
cd /opt/software
mkdir -p /opt/debezium/connector
tar -zxvf debezium-connector-mysql-1.7.2.Final-plugin.tar.gz -C /opt/debezium/connector/
```

2. 配置Mysql Connector插件

```bash
cd /opt/kafka_2.13-3.1.0/config
mv connect-distributed.properties connect-distributed.properties.bak
cat connect-distributed.properties.bak | grep -v "#" | grep -v "^$" > connect-distributed.properties

bootstrap.servers=192.168.3.200:9092
group.id=connect-mysql
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false
offset.storage.topic=connect-mysql-status
offset.storage.replication.factor=2
config.storage.topic=connect-configs
config.storage.replication.factor=1
status.storage.topic=connect-status
status.storage.replication.factor=2
offset.flush.interval.ms=10000
plugin.path=/opt/debezium/connector
```

!> 需要提前创建好topic

```bash
bin/kafka-topics.sh --create --topic connect-mysql-status --bootstrap-server 192.168.3.200:9092
bin/kafka-topics.sh --create --topic connect-configs --bootstrap-server 192.168.3.200:9092
bin/kafka-topics.sh --create --topic connect-status --bootstrap-server 192.168.3.200:9092
```

?> 配置日志压缩策略

```bash
vi /opt/kafka_2.13-3.1.0/config/server.properties

#1、是否开启日志压缩
log.cleaner.enable=true
#2、启用日志压缩策略
log.cleanup.policy=compact
```

### 1.3 启动服务

```bash
cd /opt/kafka_2.13-3.1.0/
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &   # 启动zookeeper
nohup bin/kafka-server-start.sh config/server.properties &			# 启动kafka
bin/connect-distributed.sh -daemon config/connect-distributed.properties  # 启动connector
```

### 1.4 连接器配置

```bash 
{
  "name": "xzh-mysql-connector",	# 连接器名字
  "config": {  
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "192.168.3.200",  
    "database.port": "3306",
    "database.user": "root",
    "database.password": "123456",
    "database.server.id": "1",  
    "database.server.name": "bigdata",	# 服务器名，会成为topic的前缀
    "database.include.list": "test",	# 要监控的数据库列表，多个逗号
    "database.history.kafka.bootstrap.servers": "192.168.3.200:9092",  
    "database.history.kafka.topic": "schema-changes.inventory"  
  }
}
```

1.	检测kafka连接器的服务状态

```bash
curl -H "Accept:application/json" 192.168.3.200:8083/
```

2.	检查向Kafka Connect注册的连接器列表

```bash
curl -H "Accept:application/json" 192.168.3.200:8083/connectors/
```

3. 注册连接器

```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 192.168.3.200:8083/connectors/ -d '{"name":"xzh-mysql-connector","config":{"connector.class":"io.debezium.connector.mysql.MySqlConnector","database.hostname":"192.168.3.200","database.port":"3306","database.user":"root","database.password": "123456","database.server.id":"1","database.server.name":"bigdata","database.include.list":"test","database.history.kafka.bootstrap.servers":"192.168.3.200:9092","database.history.kafka.topic":"schema-changes.inventory"}}'
```

4. 删除连接器

```bash
curl -i -X DELETE -H "Accept:application/json" -H "Content-Type:application/json" 192.168.3.200:8083/connectors/xzh-mysql-connector
```

5. 获取connector具体信息

```bash
curl -i -X GET 192.168.3.200:8083/connectors/xzh-mysql-connector
```

5. 获取connector的配置信息

```bash
curl -i -X GET 192.168.3.200:8083/connectors/xzh-mysql-connector/config
```

6.  获取connector的状态信息

```bash
curl -i -X GET 192.168.3.200:8083/connectors/xzh-mysql-connector/status
```

7. 暂停connector

```bash
curl -i -X PUT 192.168.3.200:8083/connectors/xzh-mysql-connector/pause
```

8. 恢复connector

```bash
curl -i -X PUT 192.168.3.200:8083/connectors/xzh-mysql-connector/resume
```

9. 获取connector创建的所有topic

```bash
curl -i -X GET 192.168.3.200:8083/connectors/xzh-mysql-connector/topics
```

10. 清空connector的topic

```bash
curl -i -X PUT 192.168.3.200:8083/connectors/xzh-mysql-connector/topics/reset
```

11. 获取connector的任务列表

```bash
curl -i -X GET 192.168.3.200:8083/connectors/xzh-mysql-connector/tasks
```

12. 获取connector的某个任务的状态

```bash
curl -i -X GET 192.168.3.200:8083/connectors/xzh-mysql-connector/tasks/{taskid}/status
```

### 1.5 测试

topic名字就是: 服务器名.数据库名.表名[bigdata.test.stu]，启动kafka-eagle进行查看

```bash
bin/kafka-console-consumer.sh --topic bigdata.test.stu --from-beginning --bootstrap-server 192.168.3.200:9092 # 监控变化
```


## 2. PostgreSQL

### 2.1 PostgreSQL准备

1. 安装PostgreSQL14

```bash
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install -y postgresql14-server
sudo /usr/pgsql-14/bin/postgresql-14-setup initdb
sudo systemctl enable postgresql-14
sudo systemctl start postgresql-14
```

2. 修改配置

```bash
vi /var/lib/pgsql/14/data/postgresql.conf

listen_addresses = '*'
wal_level = logical
max_wal_senders = 2
max_replication_slots = 1
```

## 2. SQL Server

## 2. MongoDB