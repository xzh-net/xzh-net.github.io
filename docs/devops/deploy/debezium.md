# Debezium 1.7.2

官方网站：https://debezium.io/

## 1. 安装

### 1.1 下载

- https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/1.7.2.Final/debezium-connector-mysql-1.7.2.Final-plugin.tar.gz
- https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/1.7.2.Final/debezium-connector-postgres-1.7.2.Final-plugin.tar.gz
- https://repo1.maven.org/maven2/io/debezium/debezium-connector-sqlserver/1.7.2.Final/debezium-connector-sqlserver-1.7.2.Final-plugin.tar.gz
- https://repo1.maven.org/maven2/io/debezium/debezium-connector-mongodb/1.7.2.Final/debezium-connector-mongodb-1.7.2.Final-plugin.tar.gz

### 1.2 配置

1. kafak插件配置

```bash
cd /opt/kafka_2.13-3.1.0/config
mv connect-distributed.properties connect-distributed.properties.bak
cat connect-distributed.properties.bak | grep -v "#" | grep -v "^$" > connect-distributed.properties

vi connect-distributed.properties
# 修改配置
bootstrap.servers=192.168.3.200:9092
group.id=connect-xuzhihao
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false
offset.storage.topic=connect-xuzhihao-status
offset.storage.replication.factor=2
config.storage.topic=connect-configs
config.storage.replication.factor=1
status.storage.topic=connect-status
status.storage.replication.factor=2
offset.flush.interval.ms=10000
plugin.path=/opt/debezium/connector
```

2. 创建topic

```bash
bin/kafka-topics.sh --create --topic connect-xuzhihao-status --bootstrap-server 192.168.3.200:9092
bin/kafka-topics.sh --create --topic connect-configs --bootstrap-server 192.168.3.200:9092
bin/kafka-topics.sh --create --topic connect-status --bootstrap-server 192.168.3.200:9092
```

3. 配置日志压缩策略

```bash
vi /opt/kafka_2.13-3.1.0/config/server.properties

log.cleaner.enable=true     # 开启日志压缩
log.cleanup.policy=compact  # 用日志压缩
```


### 1.3 Connector API

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

## 2. MySQL

### 2.1 MySQL准备

1. 修改配置

```bash
vi /etc/my.cnf

server_id=1
log_bin=mysql-bin
binlog_format=ROW
```

重启服务校验binlog是否开启成功

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
insert into stu values(1, 'zs', 18);
update stu set age=19 where id=1;
delete from stu where id=1;
```

### 2.2 安装Mysql Connector

上传解压

```bash
cd /opt/software
mkdir -p /opt/debezium/connector
tar -zxvf debezium-connector-mysql-1.7.2.Final-plugin.tar.gz -C /opt/debezium/connector/
```

### 2.3 启动服务

```bash
cd /opt/kafka_2.13-3.1.0/
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &   # 启动zookeeper
nohup bin/kafka-server-start.sh config/server.properties &			# 启动kafka
bin/connect-distributed.sh -daemon config/connect-distributed.properties  # 启动connector
```

### 2.4 注册连接器

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

```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 192.168.3.200:8083/connectors/ -d '{"name":"xzh-mysql-connector","config":{"connector.class":"io.debezium.connector.mysql.MySqlConnector","database.hostname":"192.168.3.200","database.port":"3306","database.user":"root","database.password": "123456","database.server.id":"1","database.server.name":"bigdata","database.include.list":"test","database.history.kafka.bootstrap.servers":"192.168.3.200:9092","database.history.kafka.topic":"schema-changes.inventory"}}'

curl -i -X GET 192.168.3.200:8083/connectors/xzh-mysql-connector/topics
curl -i -X DELETE -H "Accept:application/json" -H "Content-Type:application/json" 192.168.3.200:8083/connectors/xzh-mysql-connector
```

### 2.5 测试

topic名字就是: 服务器名.数据库名.表名，启动kafka-eagle进行查看

```bash
bin/kafka-console-consumer.sh --topic bigdata.test.stu --from-beginning --bootstrap-server 192.168.3.200:9092 # 监控变化
```


## 3. PostgreSQL

### 3.1 PostgreSQL准备

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
# 修改
listen_addresses = '*'
wal_level = logical
max_wal_senders = 2       # 设置同时运行的WAL发送进程最大数量
max_replication_slots = 2 # 设置同步的已定义复制槽的最大数

vi /var/lib/pgsql/14/data/pg_hba.conf
# 添加
host    all             all             0.0.0.0/0            scram-sha-256
host    replication     all             0.0.0.0.0               trust

# 重启数据库
systemctl restart postgresql-14
```

3. 准备测试库和表

```bash
sudo -i -u postgres
psql
create user "sonar" with password '123456'; # 创建新用户
create database "sonardb" template template1 owner "sonar"; # 创建用户数据库
grant all privileges on database "sonardb" to "sonar";  # 把库的所有权赋给用户
alter user sonar replication superuser;  # 给用户添加application和权限
exit

sudo passwd -d postgres   # 删除postgres用户密码
sudo passwd postgres      # 重新设置密码
su - postgres
psql -d sonardb
create table product(id int, name varchar(25), PRIMARY KEY (id));
insert into product values(1, 'lisi');
update product set name='zhangsan' where id=1;
delete from product where id=1;
SELECT * FROM pg_replication_slots;
```

### 3.2 安装PostgreSQL Connector

上传解压

```bash
cd /opt/software
mkdir -p /opt/debezium/connector
tar -zxvf debezium-connector-postgres-1.7.2.Final-plugin.tar.gz -C /opt/debezium/connector/
```

### 3.3 启动服务

```bash
cd /opt/kafka_2.13-3.1.0/
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &   # 启动zookeeper
nohup bin/kafka-server-start.sh config/server.properties &			    # 启动kafka
bin/connect-distributed.sh -daemon config/connect-distributed.properties  # 启动connector
```

### 3.4 注册连接器

```bash
{
    "name":"xzh-psql-connector",
    "config":{
        "connector.class":"io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname":"192.168.3.200",
        "database.port":"5432",
        "database.dbname":"sonardb",
        "database.user":"sonar",
        "database.password":"123456",
        "database.server.name":"pgsql4",
        "table.whitelist": "public.product",
        "plugin.name":"pgoutput"
    }
}

```

!> 如果之前调试MySQL连接器，主题必须重建，否则新的连接器会无法获取数据

```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{"name":"xzh-psql-connector","config":{"connector.class":"io.debezium.connector.postgresql.PostgresConnector","database.hostname":"192.168.3.200","database.port":"5432","database.dbname":"sonardb","database.user":"sonar","database.password":"123456","database.server.name":"pgsql4","table.whitelist":"public.product","plugin.name":"pgoutput"}}'

curl -i -X GET 192.168.3.200:8083/connectors/xzh-psql-connector/topics
curl -i -X DELETE -H "Accept:application/json" -H "Content-Type:application/json" 192.168.3.200:8083/connectors/xzh-psql-connector
```

### 3.5 测试

每个被监控的表在Kafka都会对应一个topic，topic的命名规范是<database.server.name>.<schema>.<table>

```bash
bin/kafka-console-consumer.sh --topic pgsql4.public.product --from-beginning --bootstrap-server 192.168.3.200:9092 # 监控变化
```

## 4. SQL Server

## 5. MongoDB