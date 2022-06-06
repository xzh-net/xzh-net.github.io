# Docker Hub

## 1. Container

### portainer-ce

```bash
docker volume create portainer_data

docker run -d -p 8000:8000 -p 9443:9443 --name portainer \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:2.11.0
```

```bash
vim /usr/lib/systemd/system/docker.service #编辑文件
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock #修改参数
systemctl daemon-reload #加载docker守护线程
systemctl restart docker #重启docker
```

### Nginx

```bash
docker run -p 80:80 --name nginx \
-v /mydata/nginx/html:/usr/share/nginx/html \
-v /mydata/nginx/logs:/var/log/nginx  \
-d nginx:1.20.2
```

```bash
docker container cp nginx:/etc/nginx /mydata/nginx/ # 将容器内的配置文件拷贝到指定目录
mv /mydata/nginx/nginx /mydata/nginx/conf           # 修改文件
docker rm -f nginx
```

启动Nginx服务：
```bash
docker run -p 80:80 -p 443:443 --name nginx \
-v /mydata/nginx/html:/usr/share/nginx/html \
-v /mydata/nginx/logs:/var/log/nginx  \
-v /mydata/nginx/conf:/etc/nginx \
-d nginx:1.20.2
```

### Rancher

```bash
mkdir -p /mydata/rancher_home/rancher
mkdir -p /mydata/rancher_home/auditlog

docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 \
  -v /mydata/rancher_home/rancher:/var/lib/rancher \
  -v /mydata/rancher_home/auditlog:/var/log/auditlog \
  --name rancher rancher/rancher 
```

重置密码
```bash
docker exec -it rancher reset-password
```

## 2. RDBMS

### MySQL


```bash
docker pull mysql:5.7

docker run -p 3306:3306 --name mysql \
-v /mydata/mysql/log:/var/log/mysql \
-v /mydata/mysql/data:/var/lib/mysql \
-v /mydata/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root  \
-d mysql:5.7

docker cp /mydata/mall.sql mysql:/  # sql拷贝到容器中
docker exec -it mysql /bin/bash
mysql -uroot -proot --default-character-set=utf8
create database mall character set utf8
use mall;
source /mall.sql;
grant all privileges on *.* to 'root' @'%' identified by '123456';
```

### PostgreSQL

```bash
docker volume create pgdata  # 创建本地卷，数据卷可以在容器之间共享和重用，默认会一直存在，即使容器被删除
docker volume inspect pgdata # 查看数据卷的本地位置

docker run  -p 5432:5432 --name postgres2 \
    -e POSTGRES_USER=postgres \
    -e POSTGRES_PASSWORD=123456 \
    -e POSTGRES_DB=testdb \
    -e TZ=PRC  \
    -v pgdata:/var/lib/postgresql/data \
    -d postgres:12.4

docker exec -it postgres2 /bin/bash
/var/lib/postgresql/data   #  镜像的data目录
/usr/lib/postgresql/12/bin #  工具目录
psql -Upostgres # 连接数据库
```

### Oracle

```bash
docker pull wnameless/oracle-xe-11g
docker pull wnameless/oracle-xe-11g-r2

docker run --name oracle-xe-11g -d -v /mydata/oracle_data:/data/oracle_data -p 49160:22 -p 49161:1521 -e ORACLE_ALLOW_REMOTE=true wnameless/oracle-xe-11g:14.04.4

port: 49161
sid: xe
username: system
password: oracle
system: oracle
sys: oracle

docker exec -it oracle-xe-11g /bin/bash

cd /u01/app/oracle
mkdir xuzhihao
chmod 777 xuzhihao
su oracle
cd $ORACLE_HOME
bin/sqlplus / as sysdba

create tablespace xuzhihao datafile '/u01/app/oracle/xuzhihao/xuzhihao.dbf' size 100M;
create user xuzhihao identified by 123456 default tablespace xuzhihao;
grant connect,resource to xuzhihao;
grant dba to xuzhihao;  #  授予dba权限后，这个用户能操作所有用户的表

```

## 3. NoSQL

### Memcached

```bash
docker pull memcached:1.6.12
docker run -dit --name memcached -m 128m -c 16382 -p 11211:11211 -d memcached:1.6.12
```

可视化
```bash
docker run -dit -p 8080:8080 -e MEMADMIN_USERNAME='admin' -e MEMADMIN_PASSWORD='admin' -e MEMCACHED_HOST='172.17.17.200' -e MEMCACHED_PORT='11211' vesica/memadmin:latest
# 启动后报错进入容器，文件最后添加一行
vim /etc/apache2/apache2.conf
ServerName localhost:80
# 然后保存并重启Apache2
service apache2 restart
```

### Redis

- 单机

```bash
docker run -p 6379:6379 --name redis \
-v /mydata/redis/data:/data \
-d redis:5 redis-server --appendonly yes --requirepass "123456"
```

- 布隆过滤器（6.0.9）

```bash
docker run -dit -p 6379:6379 --name redis-redisbloom -d --restart=always -e TZ="Asia/Shanghai" redislabs/rebloom:2.2.8 --requirepass  "redis6379"
```

- 监控

```bash
docker run --name redis-stat --link some-redis:redis -p 8080:63790 -d insready/redis-stat --server redis          # 容器内部自连接
docker run --name redis-stat -p 8080:63790 -d insready/redis-stat --server 192.168.3.200:6379                     # 远程服务器
docker run --name redis-stat -p 8080:63790 -d insready/redis-stat --server 192.168.3.200:6379 192.168.3.201:6379  # 远程集群或单机
```

- prometheus监控

```bash
docker pull oliver006/redis_exporter:v1.28.0
docker run -d --name redis_exporter16379 -p 16379:9121 oliver006/redis_exporter:v1.28.0 --redis.addr redis://172.17.17.191:16379 --redis.password 'redis16379'
```

### MongoDB

```bash
docker pull mongo:4.4.6

docker run -p 27017:27017 --name mongo \
-v /mydata/mongo/db:/data/db \
-d mongo:4.4.6
```

### CouchDB 

```bash
docker run -p 5984:5984 --name my-couchdb -e COUCHDB_USER=admin -e COUCHDB_PASSWORD=admin -v /home/mydata/couchdb/data:/opt/couchdb/data -d couchdb:3.2
docker run -d -p 8800:8000 --link=my-couchdb --name fauxton 3apaxicom/fauxton sh -c 'fauxton -c http://192.168.3.200:5984'  # 可视化
```

### InfluxDB

- 1.8

```bash
docker run -d -p 8086:8086 \
      -v /mydata/influxdb1:/var/lib/influxdb \
      --name influxdb1 \
      influxdb:1.8
```

- 2.0.6

```bash
docker run -d -p 8086:8086 \
      -v /mydata/influxdb2/data:/var/lib/influxdb2 \
      -v /mydata/influxdb2/config:/etc/influxdb2 \
      -e DOCKER_INFLUXDB_INIT_MODE=setup \
	    -e DOCKER_INFLUXDB_INIT_USERNAME=my-user \
      -e DOCKER_INFLUXDB_INIT_PASSWORD=my-password \
      -e DOCKER_INFLUXDB_INIT_ORG=org \
      -e DOCKER_INFLUXDB_INIT_BUCKET=bucket \
	    --name influxdb2 \
      influxdb:2.0.6
```

### HBase

```bash
docker pull harisekhon/hbase:2.1
docker run -dti --name hbase -p 16010:16010 harisekhon/hbase:2.1
docker exec -it hbase /bin/bash
hbase shell
```

### TDengine

```bash
docker run -d -p 6041:6041 \
	-v /mydata/taos/conf:/etc/taos \
	-v /mydata/taos/data:/var/lib/taos \
	-v /mydata/taos/logs:/var/log/taos \
	--name tdengine 
	tdengine:2.0.19.1
```

## 4. Middleware

### MyCat

vi dockerfile

```conf
FROM centos:7 
RUN echo "root:root" | chpasswd
RUN yum -y install net-tools

# install java
ADD http://mirrors.linuxeye.com/jdk/jdk-8u221-linux-x64.tar.gz /usr/local/
RUN cd /usr/local && tar -zxvf jdk-8u221-linux-x64.tar.gz && ls -lna

ENV JAVA_HOME /usr/local/jdk1.8.0_221
ENV CLASSPATH ${JAVA_HOME}/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV PATH $PATH:${JAVA_HOME}/bin

#install mycat

ADD http://dl.mycat.io/1.6.7.3/20190828135747/Mycat-server-1.6.7.3-release-20190828135747-linux.tar.gz /usr/local
RUN cd /usr/local && tar -zxvf Mycat-server-1.6.7.3-release-20190828135747-linux.tar.gz && ls -lna

#download mycat-ef-proxy
#RUN mkdir -p /usr/local/proxy
#ADD https://github.com/LonghronShen/mycat-docker/releases/download/1.6/MyCat-Entity-Framework-Core-Proxy.1.0.0-alpha2-netcore100.tar.gz /usr/local/proxy
#RUN cd /usr/local/proxy && tar -zxvf MyCat-Entity-Framework-Core-Proxy.1.0.0-alpha2-netcore100.tar.gz && ls -lna && sed -i -e 's#C:\\\\mycat#/usr/local/mycat#g' config.json

VOLUME /usr/local/mycat/conf

EXPOSE 8066 9066
#EXPOSE 7066

CMD /usr/local/mycat/bin/mycat console
```

```bash
Docker  build –f  dockerfile –t mycat  .                                          # 生成镜像
docker  run -dti --network mysql -p 8066:8066 -p 9066:9066 --name mycat mycat     # network根据环境调整
docker  cp  mycat:/mydata/mycat/conf/  /usr/local/mycat/conf/                     # 挂载配置文件
docker  rm -f  mycat
```

server.xml 配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- - - Licensed under the Apache License, Version 2.0 (the "License"); 
   - you may not use this file except in compliance with the License. - You 
   may obtain a copy of the License at - - http://www.apache.org/licenses/LICENSE-2.0 
   - - Unless required by applicable law or agreed to in writing, software - 
   distributed under the License is distributed on an "AS IS" BASIS, - WITHOUT 
   WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. - See the 
   License for the specific language governing permissions and - limitations 
   under the License. -->
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
   <system>
   <property name="nonePasswordLogin">0</property> <!-- 0为需要密码登陆、1为不需要密码登陆 ,默认为0，设置为1则需要指定默认账户-->
   <property name="useHandshakeV10">1</property>
   <property name="useSqlStat">0</property>  <!-- 1为开启实时统计、0为关闭 -->
   <property name="useGlobleTableCheck">0</property>  <!-- 1为开启全加班一致性检测、0为关闭 -->
      <property name="sqlExecuteTimeout">300</property>  <!-- SQL 执行超时 单位:秒-->
      <property name="sequnceHandlerType">2</property>
      <!--<property name="sequnceHandlerPattern">(?:(\s*next\s+value\s+for\s*MYCATSEQ_(\w+))(,|\)|\s)*)+</property>-->
      <!--必须带有MYCATSEQ_或者 mycatseq_进入序列匹配流程 注意MYCATSEQ_有空格的情况-->
      <property name="sequnceHandlerPattern">(?:(\s*next\s+value\s+for\s*MYCATSEQ_(\w+))(,|\)|\s)*)+</property>
   <property name="subqueryRelationshipCheck">false</property> <!-- 子查询中存在关联查询的情况下,检查关联字段中是否有分片字段 .默认 false -->
      <!--  <property name="useCompression">1</property>--> <!--1为开启mysql压缩协议-->
        <!--  <property name="fakeMySQLVersion">5.6.20</property>--> <!--设置模拟的MySQL版本号-->
   <!-- <property name="processorBufferChunk">40960</property> -->
   <!-- 
   <property name="processors">1</property> 
   <property name="processorExecutor">32</property> 
    -->
        <!--默认为type 0: DirectByteBufferPool | type 1 ByteBufferArena | type 2 NettyBufferPool -->
      <property name="processorBufferPoolType">0</property>
      <!--默认是65535 64K 用于sql解析时最大文本长度 -->
      <!--<property name="maxStringLiteralLength">65535</property>-->
      <!--<property name="sequnceHandlerType">0</property>-->
      <!--<property name="backSocketNoDelay">1</property>-->
      <!--<property name="frontSocketNoDelay">1</property>-->
      <!--<property name="processorExecutor">16</property>-->
      <!--
         <property name="serverPort">8066</property> <property name="managerPort">9066</property> 
         <property name="idleTimeout">300000</property> <property name="bindIp">0.0.0.0</property>
         <property name="dataNodeIdleCheckPeriod">300000</property> 5 * 60 * 1000L; //连接空闲检查
         <property name="frontWriteQueueSize">4096</property> <property name="processors">32</property> -->
      <!--分布式事务开关，0为不过滤分布式事务，1为过滤分布式事务（如果分布式事务内只涉及全局表，则不过滤），2为不过滤分布式事务,但是记录分布式事务日志-->
      <property name="handleDistributedTransactions">0</property>
      
         <!--
         off heap for merge/order/group/limit      1开启   0关闭
      -->
      <property name="useOffHeapForMerge">0</property>

      <!--单位为m-->
        <property name="memoryPageSize">64k</property>

      <!--单位为k-->
      <property name="spillsFileBufferSize">1k</property>

      <property name="useStreamOutput">0</property>

      <!--单位为m-->
      <property name="systemReserveMemorySize">384m</property>


      <!--是否采用zookeeper协调切换  -->
      <property name="useZKSwitch">false</property>

      <!-- XA Recovery Log日志路径 -->
      <!--<property name="XARecoveryLogBaseDir">./</property>-->

      <!-- XA Recovery Log日志名称 -->
      <!--<property name="XARecoveryLogBaseName">tmlog</property>-->
      <!--如果为 true的话 严格遵守隔离级别,不会在仅仅只有select语句的时候在事务中切换连接-->
      <property name="strictTxIsolation">false</property>
      
      <property name="useZKSwitch">true</property>
      
   </system>
   
   <!-- 全局SQL防火墙设置 -->
   <!--白名单可以使用通配符%或着*-->
   <!--例如<host host="127.0.0.*" user="root"/>-->
   <!--例如<host host="127.0.*" user="root"/>-->
   <!--例如<host host="127.*" user="root"/>-->
   <!--例如<host host="1*7.*" user="root"/>-->
   <!--这些配置情况下对于127.0.0.1都能以root账户登录-->
   <!--
   <firewall>
      <whitehost>
         <host host="1*7.0.0.*" user="root"/>
      </whitehost>
       <blacklist check="false">
       </blacklist>
   </firewall>
   -->
	<!-- 添加用户，及用户能访问的逻辑库  -->
    <user name="tx0001">
        <property name="password">wycqq12345</property>
        <property name="schemas">block_master_eth</property>
        <property name="readOnly">true</property>
    </user>
</mycat:server>
```

schema.xml 配置实现读写分离，定义逻辑库 ，分片，分表，
```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
    <schema name="block_master_eth" checkSQLschema="true" sqlMaxLimit="100">
        <table name="blocks" ruleRequired="false" primaryKey="block_number"  autoIncrement="false"  dataNode="eth"  />
        <table name="transactions" ruleRequired="true"  subTables="transactions$1-100"   autoIncrement="false"  dataNode="eth" rule="transactions_rule" />
    </schema>

    <dataNode name="eth"  dataHost="host"  database="block_master_eth" />

    <dataHost name="host" maxCon="2000" minCon="200" balance="3"
        dbType="mysql" dbDriver="jdbc" switchType="1">
        <heartbeat>select user()</heartbeat>
        <writeHost host="btcM1" url="jdbc:mysql://masterIP:3306/" user="root" password="wycqq123456">
            <readHost host="btcS1" url="jdbc:mysql://slaveIP:3308/" user="root" password="wycqq123456" />
        </writeHost>
    </dataHost>
</mycat:schema>
```

rule.xml 配置定义分表，分片规则
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:rule SYSTEM "rule.dtd">
<mycat:rule xmlns:mycat="http://io.mycat/">
    <tableRule name="transactions_rule">
        <rule>
            <columns>block_number</columns>
            <algorithm>t_fun_block</algorithm>
        </rule>
    </tableRule>


    <function name="t_fun_block" class="io.mycat.route.function.PartitionByPattern">
        <property name="patternValue">2000</property>
        <property name="defaultNode">0</property>
        <property name="mapFile">partition-pattern.txt</property>
    </function>
    </mycat:rule>
```


### Sharding-Proxy

/mydata/sharding_proxy/conf下创建server.yaml
```yaml
authentication:
  users:
    root:
      password: root
    sharding:
      password: sharding
      authorizedSchemas: dbgirl
props:
  acceptor.size: 16
```

创建config-dbgirl.yaml
```yaml
schemaName: dbgirl
dataSources:
  ds0:
    username: root
    password: 123456
    url: jdbc:mysql://132.126.3.141:3306/dbgirl?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
  ds1:
    username: root
    password: 123456
    url: jdbc:mysql://132.126.3.141:3306/dbgirl2?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
shardingRule:
  tables:
    girl:
      actualDataNodes: ds$->{0..1}.girl
      databaseStrategy:
        inline:
          shardingColumn: id
          algorithmExpression: ds$->{id % 2}
      keyGenerator:
        type: SNOWFLAKE
        column: id
  bindingTables:
    - girl
  defaultTableStrategy:
    none:
```

```bash
docker run -p 13308:3308 -d \
  -v /mydata/sharding_proxy/conf:/opt/sharding-proxy/conf  \
  -v /mydata/sharding_proxy/ext-lib:/opt/sharding-proxy/ext-lib  \
  -e JVM_OPTS="-Djava.awt.headless=true"  \
  --name sharding-proxy  \
  apache/sharding-proxy:latest
```


## 5. DFS

### FastDFS

```bash
docker run -dti --network=host --name tracker -v /mydata/fdfs/tracker:/var/fdfs delron/fastdfs tracker 
docker run -dti --network=host --name storage -p 8888:8888 -p 23000:23000  \
   -e TRACKER_SERVER=172.17.17.200:22122 -v /mydata/fdfs/storage:/var/fdfs delron/fastdfs storage
```

```bash
docker run --net=host --name=fastdfs -e IP=172.17.17.200 -e WEB_PORT=80 -v /mydata/fdfs:/var/local/fdfs \
	-d registry.cn-beijing.aliyuncs.com/tianzuo/fastdfs
```

### MinIO

minioadmin/minioadmin

```bash
docker pull minio/minio:RELEASE.2022-01-04T07-41-07Z

docker run -dit -p 9000:9000 -p 9001:9001 --name minio \
  -v /mnt/data:/data \
  -v /mnt/config:/root/.minio \
  minio/minio:RELEASE.2022-01-04T07-41-07Z server /data --console-address ":9001"
```

### Hadoop

```bash
docker run -dit --name hadoop-docker \
 -p 8020:8020 -p 8088:8088 -p 8040:8040 -p 8042:8042 \
 -p 50070:50070 -p 49707:49707 -p 50010:50010 -p 50075:50075 \
 -p 50090:50090 sequenceiq/hadoop-docker:2.7.0 /etc/bootstrap.sh -bash
```

```bash
docker exec -it [容器id] /bin/bash
cd /usr/local/hadoop-2.7.0/
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar grep input output 'dfs[a-z.]+'
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar pi 2 4

vi /etc/profile
export HADOOP_HOME="/usr/local/hadoop-2.7.0"
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
source /etc/profile

docker cp /root/hadoop-mapreduce-examples-2.7.0.jar [容器id]:/usr/local/hadoop-2.7.0
hadoop fs -mkdir -p /wordcount/input
hadoop fs -put a.txt /wordcount/input
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar wordcount /wordcount/input /wordcount/output
hadoop fs -cat /wordcount/output/part-r-00000
```

```lua
组件	节点	默认端口	配置	用途说明
HDFS	DataNode	50010	dfs.datanode.address	datanode服务端口，用于数据传输
HDFS	DataNode	50075	dfs.datanode.http.address	http服务的端口
HDFS	DataNode	50475	dfs.datanode.https.address	https服务的端口
HDFS	DataNode	50020	dfs.datanode.ipc.address	ipc服务的端口
HDFS	NameNode	50070	dfs.namenode.http-address	http服务的端口
HDFS	NameNode	50470	dfs.namenode.https-address	https服务的端口
HDFS	NameNode	8020	fs.defaultFS	接收Client连接的RPC端口，用于获取文件系统metadata信息。
HDFS	journalnode	8485	dfs.journalnode.rpc-address	RPC服务
HDFS	journalnode	8480	dfs.journalnode.http-address	HTTP服务
HDFS	ZKFC	8019	dfs.ha.zkfc.port	ZooKeeper FailoverController，用于NN HA
YARN	ResourceManager	8032	yarn.resourcemanager.address	RM的applications manager(ASM)端口
YARN	ResourceManager	8030	yarn.resourcemanager.scheduler.address	scheduler组件的IPC端口
YARN	ResourceManager	8031	yarn.resourcemanager.resource-tracker.address	IPC
YARN	ResourceManager	8033	yarn.resourcemanager.admin.address	IPC
YARN	ResourceManager	8088	yarn.resourcemanager.webapp.address	http服务端口
YARN	NodeManager	8040	yarn.nodemanager.localizer.address	localizer IPC
YARN	NodeManager	8042	yarn.nodemanager.webapp.address	http服务端口
YARN	NodeManager	8041	yarn.nodemanager.address	NM中container manager的端口
YARN	JobHistory Server	10020	mapreduce.jobhistory.address	IPC
YARN	JobHistory Server	19888	mapreduce.jobhistory.webapp.address	http服务端口
HBase	Master	60000	hbase.master.port	IPC
HBase	Master	60010	hbase.master.info.port	http服务端口
HBase	RegionServer	60020	hbase.regionserver.port	IPC
HBase	RegionServer	60030	hbase.regionserver.info.port	http服务端口
HBase	HQuorumPeer	2181	hbase.zookeeper.property.clientPort	HBase-managed ZK mode，使用独立的ZooKeeper集群则不会启用该端口。
HBase	HQuorumPeer	2888	hbase.zookeeper.peerport	HBase-managed ZK mode，使用独立的ZooKeeper集群则不会启用该端口。
HBase	HQuorumPeer	3888	hbase.zookeeper.leaderport	HBase-managed ZK mode，使用独立的ZooKeeper集群则不会启用该端口。
Hive	Metastore	9083	/etc/default/hive-metastore中export PORT=<port>来更新默认端口	
Hive	HiveServer	10000	/etc/hive/conf/hive-env.sh中export HIVE_SERVER2_THRIFT_PORT=<port>来更新默认端口	
ZooKeeper	Server	2181	/etc/zookeeper/conf/zoo.cfg中clientPort=<port>	对客户端提供服务的端口
ZooKeeper	Server	2888	/etc/zookeeper/conf/zoo.cfg中server.x=[hostname]:nnnnn[:nnnnn]，标蓝部分	follower用来连接到leader，只在leader上监听该端口。
ZooKeeper	Server	3888	/etc/zookeeper/conf/zoo.cfg中server.x=[hostname]:nnnnn[:nnnnn]，标蓝部分	用于leader选举的。只在electionAlg是1,2或3(默认)时需要。
```


## 6. Distributed Middleware

### Nacos
```bash
docker run --name nacos -d --network=host -p 8848:8848 -e MODE=standalone nacos/nacos-server:2.0.1

docker run --name nacos -d --network=host -p 8848:8848 -e MODE=standalone \
-e JVM_XMS=256m -e JVM_XMX=256m \
-e SPRING_DATASOURCE_PLATFORM=mysql \
-e MYSQL_SERVICE_HOST=172.17.17.137 \
-e MYSQL_SERVICE_PORT=3306 \
-e MYSQL_SERVICE_USER=root \
-e MYSQL_SERVICE_PASSWORD=root \
-e MYSQL_SERVICE_DB_NAME=nacos_config \
-e TIME_ZONE='Asia/Shanghai' \
--restart=always nacos/nacos-server:2.0.1
```

### Consul

```
docker run -d -p 8500:8500 --restart=always --name=consul consul:1.12.1 agent -server -bootstrap -ui -node=1 -client='0.0.0.0'
```


### Seata

```bash
mkdir -p /opt/seata/config/
cd /opt/seata/config
vi registry.conf

registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    application = "seata-server"
    serverAddr = "172.17.17.165:8848"
    group = "SEATA_GROUP"
    namespace = "1207ed15-5658-48db-a35a-8e3725930070"
    cluster = "default"
    username = "nacos"
    password = "nacos"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"

  nacos {
    serverAddr = "172.17.17.165:8848"
    namespace = "1207ed15-5658-48db-a35a-8e3725930070"
    group = "SEATA_GROUP"
    username = "nacos"
    password = "nacos"
  }
}

vi file.conf

store {
  mode = "db"
  db {
    datasource = "druid"
    dbType = "postgresql"
    driverClassName = "org.postgresql.Driver"
    url = "jdbc:postgresql://172.17.17.29:5432/seata"
    user = "postgres"
    password = "vjsp2020"
    minConn = 5
    maxConn = 100
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }
}
```

```bash
docker run --name seata-server -it -d  -p 8091:8091 \
-e SEATA_CONFIG_NAME=file:/root/seata/config/registry \
-e SEATA_IP=172.17.17.137 \
-v /opt/seata/config/:/root/seata/config \
--net=bridge --restart=always docker.io/seataio/seata-server:1.4.0
```

```bash
SEATA_IP        # 可选, 指定seata-server启动的IP, 该IP用于向注册中心注册时使用, 如eureka等
SEATA_PORT      # 可选, 指定seata-server启动的端口, 默认为 8091
STORE_MODE      # 可选, 指定seata-server的事务日志存储方式, 支持db ,file,redis(Seata-Server 1.3及以上版本支持), 默认是 file
SERVER_NODE     # 可选, 用于指定seata-server节点ID, 如 1,2,3..., 默认为 根据ip生成
SEATA_ENV       # 可选, 指定 seata-server 运行环境, 如 dev, test 等, 服务启动时会使用 registry-dev.conf 这样的配置
SEATA_CONFIG_NAME   # 可选, 指定配置文件位置, 如 file:/root/registry, 将会加载 /root/registry.conf 作为配置文件，
# 如果需要同时指定 file.conf文件，需要将registry.conf的config.file.name的值改为类似file:/root/file.conf：
```

### Sentinel

```bash
docker run --name sentinel -d -p 8858:8858 -d bladex/sentinel-dashboard:1.7.2
```

### Dubbo Admin

```bash
docker run -d -p 7001:7001 -e dubbo.registry.address=zookeeper://172.17.17.200:2181 \
-e dubbo.admin.root.password=root \
-e dubbo.admin.guest.password=guest \
chenchuxin/dubbo-admin
```

### Zookeeper

```bash
docker run -d -p 2181:2181 --name some-zookeeper --restart always -d zookeeper:3.7.0
```

```
docker pull zookeeper:3.7.0

mkdir /mydata/zookeeper/conf/ -p
cd /mydata/zookeeper/conf/
touch zoo.cfg

# 设置心跳时间，单位毫秒
tickTime=2000
# 存储内存数据库快照的文件夹
dataDir=/tmp/zookeeper
# 监听客户端连接的端口
clientPort=2181

docker run -p 2181:2181 --name zookeeper \
-v /mydata/zookeeper/conf/zoo.cfg:/conf/zoo.cfg \
-d zookeeper:3.7.0
```


### Zipkin 

```bash
docker run -d --name zipkin -p  9411:9411 openzipkin/zipkin:2.23
```

### SkyWalking

```bash
docker pull elasticsearch:7.6.2
docker pull apache/skywalking-oap-server:6.6.0-es7
docker pull apache/skywalking-ui:6.6.0

# 安装server 因为之前elk是compose安装,默认在mydata_default的网桥中
docker run --name oap --restart always -d \
--network mydata_default \
--restart=always \
-e TZ=Asia/Shanghai \
-p 12800:12800 \
-p 11800:11800 \
--link elasticsearch:es \
-e SW_STORAGE=elasticsearch \
-e SW_STORAGE_ES_CLUSTER_NODES=es:9200 \
apache/skywalking-oap-server:6.6.0-es7

# 安装ui
docker run -d --name skywalking-ui \
--network mydata_default \
--restart=always \
-e TZ=Asia/Shanghai \
-p 8088:8080 \
--link oap:oap \
-e SW_OAP_ADDRESS=oap:12800 \
apache/skywalking-ui:6.6.0
```

下载源码包，下面会用到agent 
> https://archive.apache.org/dist/skywalking/6.6.0/apache-skywalking-apm-6.6.0.tar.gz

```bash
java -jar skywalking_springboot.jar # 原启动方式
java -javaagent:/home/mydata/app_skywalking/apache-skywalking-apm-bin/agent/skywalking-agent.jar -Dskywalking.agent.service_name=springboot -Dskywalking.collector.backend_service=127.0.0.1:11800 -jar /home/mydata/app_skywalking/skywalking_springboot.jar
```

## 7. Message Middleware

### ActiveMQ 

admin/admin
```bash
docker run -d --name activemq -p 61616:61616 -p 8161:8161 webcenter/activemq:5.14.3
```

### RabbitMQ

- 3.7.15

```bash
docker pull rabbitmq:3.7.15
docker run -p 5672:5672 -p 15672:15672 -p 1883:1883 -p 15675:15675 --name rabbitmq -d rabbitmq:3.7.15
# 插件
wget https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/v3.8.0/rabbitmq_delayed_message_exchange-3.8.0.ez
docker cp rabbitmq_delayed_message_exchange-3.8.0.ez rabbitmq:/plugins/
docker exec -it rabbitmq /bin/bash
rabbitmq-plugins enable rabbitmq_mqtt
rabbitmq-plugins enable rabbitmq_management
rabbitmq-plugins enable rabbitmq_web_mqtt
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

- 3.9

```bash
docker run -dit --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.9-management
```

### RocketMQ

```bash
# 创建目录
mkdir -p /mydata/rocketmq/data/namesrv/logs /root/rocketmq/data/namesrv/store /mydata/rocketmq/conf /mydata/rocketmq/data/broker/logs /mydata/rocketmq/data/broker/stor
```

```conf
# 进入 /mydata/rocketmq/conf 创建 broker.conf
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
brokerIP1 = 172.17.17.200
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

拉取镜像
```bash
docker pull rocketmqinc/rocketmq:4.4.0
docker pull apacherocketmq/rocketmq-dashboard:1.0.0
```

namesrv
```bash
docker run -d -p 9876:9876 -v /mydata/rocketmq/data/namesrv/logs:/root/logs -v /mydata/rocketmq/data/namesrv/store:/root/store --name rmqnamesrv -e "MAX_POSSIBLE_HEAP=100000000" rocketmqinc/rocketmq:4.4.0 sh mqnamesrv
```

broker
```
docker run -d -p 10911:10911 -p 10909:10909 -v  /mydata/rocketmq/data/broker/logs:/root/logs -v  /mydata/rocketmq/data/broker/store:/root/store -v  /mydata/rocketmq/conf/broker.conf:/opt/rocketmq-4.4.0/conf/broker.conf --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" -e "MAX_POSSIBLE_HEAP=200000000" rocketmqinc/rocketmq:4.4.0 sh mqbroker -c /opt/rocketmq-4.4.0/conf/broker.conf
```

dashboard
```bash
docker run -d --name rocketmq-dashboard -e "JAVA_OPTS=-Drocketmq.namesrv.addr=172.17.17.200:9876" -p 9080:8080 -t apacherocketmq/rocketmq-dashboard:1.0.0
```

### Kafka

```bash
docker run -itd --name kafka -p 9092:9092 \
  --link some-zookeeper:zookeeper \
  -e KAFKA_BROKER_ID=0 \
  -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.17.17.200:9092 \
  -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
  -t wurstmeister/kafka
```

```bash
docker run -itd --name kafka-manager -p 9000:9000 -e ZK_HOSTS="172.17.17.200:2181" -e APPLICATION_SECRET=letmein sheepkiller/kafka-manager
```

### EMQ

```bash
docker run -d --name emqx -p 1883:1883 -p 8081:8081 -p 8083:8083 -p 8084:8084 -p 8883:8883 -p 18083:18083 emqx/emqx:4.3.10        # 开源版
docker run -d --name emqx-ee -p 1883:1883 -p 8081:8081 -p 8083:8083 -p 8084:8084 -p 8883:8883 -p 18083:18083 emqx/emqx-ee:4.2.9   # 企业版
```

### Pulsar


## 8. Elastic Stack

### Elasticsearch

```bash
docker pull elasticsearch:7.6.2
```

- 修改虚拟内存区域大小，否则会因为过小而无法启动:

```bash
sysctl -w vm.max_map_count=262144
```

- 使用如下命令启动Elasticsearch服务：

```bash
docker run -p 9200:9200 -p 9300:9300 --name elasticsearch \
-e "discovery.type=single-node" \
-e "cluster.name=elasticsearch" \
-v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \
-d elasticsearch:7.6.2
```

- 启动时会发现`/usr/share/elasticsearch/data`目录没有访问权限，只需要修改`/mydata/elasticsearch/data`目录的权限，再重新启动即可；

```bash
chmod 777 /mydata/elasticsearch/data/
```

- 安装中文分词器IKAnalyzer，并重新启动：

```bash
docker exec -it elasticsearch /bin/bash
#此命令需要在容器中运行
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip
docker restart elasticsearch
http://192.168.3.200:9200/_cat/plugins
```

- 安装elasticsearch-head插件

```bash
docker run -d -p 9100:9100 docker.io/mobz/elasticsearch-head:5
```

elasticsearch.yml，在文件末尾加入以下配置

```yml
http.cors.enabled: true
http.cors.allow-origin: "*"
```


### Logstash

```bash
docker pull logstash:7.6.2
```

- 创建logstash.conf文件
```
input {
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4560
    codec => json_lines
    type => "debug"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4561
    codec => json_lines
    type => "error"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4562
    codec => json_lines
    type => "business"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4563
    codec => json_lines
    type => "record"
  }
}
filter{
  if [type] == "record" {
    mutate {
      remove_field => "port"
      remove_field => "host"
      remove_field => "@version"
    }
    json {
      source => "message"
      remove_field => ["message"]
    }
  }
}
output {
  elasticsearch {
    hosts => "es:9200"
    index => "mall-%{type}-%{+YYYY.MM.dd}"
  }
}
```

- 创建`/mydata/logstash`目录，并将Logstash的配置文件`logstash.conf`拷贝到该目录；

```bash
mkdir /mydata/logstash
```

- 使用如下命令启动Logstash服务；

```bash
docker run --name logstash -p 4560:4560 -p 4561:4561 -p 4562:4562 -p 4563:4563 \
--link elasticsearch:es \
-v /mydata/logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
-d logstash:7.6.2
```

- 进入容器内部，安装`json_lines`插件。

```bash
docker exec -it logstash /bin/bash
cd /usr/share/logstash/bin
logstash-plugin install logstash-codec-json_lines
```

### Kibana

- 访问地址：http://192.168.3.200:5601

```bash
docker pull kibana:7.6.2

docker run --name kibana -p 5601:5601 \
--link elasticsearch:es \
-e "elasticsearch.hosts=http://es:9200" \
-d kibana:7.6.2
```


## 9. DevOps

### GitLab

```bash
docker pull gitlab/gitlab-ce:12.4.2-ce.0
mkdir -p /home/gitlab/{config,logs,data}  # 创建目录

docker run -d  \
-p 443:443 \
-p 80:80 \
-p 222:22 \
--name gitlab \
--restart always \
-v /home/gitlab/config:/etc/gitlab \
-v /home/gitlab/logs:/var/log/gitlab \
-v /home/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce:12.4.2-ce.0

# 修改配置，gitlab.rb文件内容默认全是注释
vi /home/gitlab/config/gitlab.rb
# 配置ssh协议所使用的访问地址和端口
gitlab_rails['gitlab_ssh_host'] = '172.17.17.196'
gitlab_rails['gitlab_shell_ssh_port'] = 222

docker restart gitlab
```


### Nexus3

```bash
docker pull sonatype/nexus3:3.36.0
mkdir -p /home/mvn/nexus-data  && chown -R 200 /home/mvn/nexus-data
docker run -d -p 8081:8081 --name nexus -v /home/mvn/nexus-data:/nexus-data sonatype/nexus3:3.36.0
```

### Harbor

> wget https://github.com/goharbor/harbor/releases/download/v2.0.1/harbor-offline-installer-v2.0.1.tgz

```shell
tar xf harbor-offline-installer-v2.0.1.tgz
mkdir /opt/harbor
mv harbor/* /opt/harbor
cd /opt/harbor
# 复制配置文件
cp harbor.yml.tmpl harbor.yml
# 编辑
vi harbor.yml
```

```yml
#修改配置文件(如果不用https就注释掉https的几项，我是用的http就注释掉了https的，其他几项修改为自己的信息)
···
hostname: 192.168.3.200
···
# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 88
···
# https related config
#https:
  # https port for harbor, default is 443
  #port: 443
  # The path of cert and key files for nginx
  #certificate: /your/certificate/path
  #private_key: /your/private/key/path
····
harbor_admin_password: Harbor12345
···
data_volume: /data
···
```

安装

```shell
./install.sh 
docker-compose up -d #启动
docker-compose stop #停止
docker-compose restart #重新启动
```

默认账户密码：admin/Harbor12345


### SonarQube

```bash
docker run -d --name sonarqube -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true -p 9000:9000 sonarqube:8.6-community #H2默认存储

#postgres外挂存储
docker run -d --name sonarqube \
    --link postgres2 \
    -p 9000:9000 \
    -e sonar.jdbc.url=jdbc:postgresql://172.17.17.80:5432/sonardb \
    -e sonar.jdbc.username=sonar \
    -e sonar.jdbc.password=123456 \
    -v /data/sonarqube/sonarqube_extensions:/opt/sonarqube/extensions \
    -v /data/sonarqube/sonarqube_logs:/opt/sonarqube/logs \
    -v /data/sonarqube/sonarqube_data:/opt/sonarqube/data \
    sonarqube:8.6-community
```

### Jenkins

```bash
docker pull jenkins/jenkins:lts
docker pull jenkins/jenkins

mkdir -p /data/jenkins_home/
chown -R 1000:1000 /data/jenkins_home/

docker run -d --name jenkins -p 8888:8080 -p 50000:50000 -v /data/jenkins_home:/var/jenkins_home jenkins/jenkins

cd /data/jenkins_home
vi hudson.model.UpdateCenter.xml
# http://mirror.xmission.com/jenkins/updates/update-center.json

```

### Prometheus

```bash
docker pull prom/prometheus
docker pull prom/node-exporter
docker pull grafana/grafana
```

docker-compose-prometheus.yml
```yml
version: '2'
services:
####################prometheus###############
  prometheus:
    image: "prom/prometheus"
    hostname: prometheus
    container_name: prometheus
    ports:
      - '9090:9090'
    volumes:
      - /mydata/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    restart: always

###############node-exporter###############
  node-exporter:
    image: "prom/node-exporter"
    hostname: node-exporter
    container_name: node-exporter
    ports:
      - '9100:9100'
    volumes:
      - /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    restart: always
    network_mode: host
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
```

prometheus.yml
```yml
global:
  scrape_interval:     60s
  evaluation_interval: 60s
 
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus
 
  - job_name: linux
    static_configs:
      - targets: ['172.17.17.201:9100']
```

```bash
docker-compose -f docker-compose-prometheus.yml up -d  
```

### Grafana

```bash
docker run -d -p 3000:3000 --name grafana grafana/grafana
```

admin:admin


## 10. Instant Message

### SRS

```bash
docker pull ossrs/srs:v4.0.187
# 默认安装
docker run --rm -p 1935:1935 -p 1985:1985 -p 8080:8080 \
    -v /home/srs-4.0.187/trunk/conf/srs.conf:/usr/local/srs/conf/srs.conf \
    ossrs/srs:v4.0.187
```

```bash
# 创建自定义网络
docker network create --driver bridge --subnet 172.0.0.0/16 xzh_network
# 查看已存在网络
docker network ls
# 创建数据目录
mkdir -p /home/docker/srs4

# 安装并启动 srs
docker run -p 1935:1935 -p 1985:1985 -p 8080:8080 \
--name srs \
ossrs/srs:v4.0.187
# 把容器中的配置文件复制出来
docker cp -a srs:/usr/local/srs/conf /home/docker/srs4/conf
# 把容器中的数据文件复制出来
docker cp -a srs:/usr/local/srs/objs /home/docker/srs4/objs
# 删除 srs 容器
docker rm -f srs

# 本地配置文件启动
docker run -p 1935:1935 -p 1985:1985 -p 8080:8080 \
--name srs \
--network xzh_network \
--ip 172.0.0.35 \
--restart=always \
-v /home/docker/srs4/conf/:/usr/local/srs/conf/ \
-v /home/docker/srs4/objs/:/usr/local/srs/objs/ \
ossrs/srs:v4.0.187 
```

http://服务器IP地址:8080

SRS自定义参考配置

```conf
listen              1935;
max_connections     1000;
srs_log_tank        file;
srs_log_file        ./objs/srs.log;
daemon              on;
http_api {
    enabled         on;
    listen          1985;
}
http_server {
    enabled         on;
    listen          8080;
    dir             ./objs/nginx/html;
	# 开启 https 支持，需要开放 8088端口
	# https {
        # enabled on;
        # listen 8088;
        # key ./conf/woniu.key;
        # cert ./conf/woniu.crt;
    # }
}
vhost __defaultVhost__ {
    # http-flv设置
    http_remux{
        enabled    on;
        mount      [vhost]/[app]/[stream].flv;
        hstrs      on;
    }
 
    # hls设置
    hls {
        enabled         on;
        hls_fragment    1;
        hls_window      2;
        hls_path        ./objs/nginx/html;
        hls_m3u8_file   [app]/[stream].m3u8;
        hls_ts_file     [app]/[stream]-[seq].ts;
    }
	
	# dvr设置
	dvr {
        enabled             off;
        dvr_path            ./objs/nginx/html/[app]/[stream]/[2006]/[01]/[02]/[timestamp].flv;
        dvr_plan            segment;
        dvr_duration        30;
        dvr_wait_keyframe   on;
    }
	
	# rtc 设置
	rtc {
		enabled     on;
		bframe      discard;
    }
	
	# SRS支持refer防盗链：检查用户从哪个网站过来的。譬如不是从公司的页面过来的人都不让看。
    refer {
        # whether enable the refer hotlink-denial.
        # default: off.
        enabled         off;
        # the common refer for play and publish.
        # if the page url of client not in the refer, access denied.
        # if not specified this field, allow all.
        # default: not specified.
        all           github.com github.io;
        # refer for publish clients specified.
        # the common refer is not overrided by this.
        # if not specified this field, allow all.
        # default: not specified.
        publish   github.com github.io;
        # refer for play clients specified.
        # the common refer is not overrided by this.
        # if not specified this field, allow all.
        # default: not specified.
        play      github.com github.io;
    }
	
	# http 回调
	http_hooks {
	
		# 事件：发生该事件时，即回调指定的HTTP地址。
		# HTTP地址：可以支持多个，以空格分隔，SRS会依次回调这些接口。
		# 数据：SRS将数据POST到HTTP接口。
		# 返回值：SRS要求HTTP服务器返回HTTP200并且response内容为整数错误码（0表示成功），其他错误码会断开客户端连接。
		
        # whether the http hooks enable.
        # default off.
        enabled         on;
        
		# 当客户端连接到指定的vhost和app时
        on_connect      http://127.0.0.1:8085/api/v1/clients http://localhost:8085/api/v1/clients;
        
		# 当客户端关闭连接，或者SRS主动关闭连接时
        on_close        http://127.0.0.1:8085/api/v1/clients http://localhost:8085/api/v1/clients;
       
		# 当客户端发布流时，譬如flash/FMLE方式推流到服务器
        on_publish      http://127.0.0.1:8085/api/v1/streams http://localhost:8085/api/v1/streams;
        
		# 当客户端停止发布流时
        on_unpublish    http://127.0.0.1:8085/api/v1/streams http://localhost:8085/api/v1/streams;
        
		# 当客户端开始播放流时
        on_play         http://127.0.0.1:8085/api/v1/sessions http://localhost:8085/api/v1/sessions;
        
		# 当客户端停止播放时。备注：停止播放可能不会关闭连接，还能再继续播放。
        on_stop         http://127.0.0.1:8085/api/v1/sessions http://localhost:8085/api/v1/sessions;
        
		# 当DVR录制关闭一个flv文件时
        on_dvr          http://127.0.0.1:8085/api/v1/dvrs http://localhost:8085/api/v1/dvrs;
		
        # 当HLS生成一个ts文件时
        on_hls          http://127.0.0.1:8085/api/v1/hls http://localhost:8085/api/v1/hls;
		
        # when srs reap a ts file of hls, call this hook,
        on_hls_notify   http://127.0.0.1:8085/api/v1/hls/[app]/[stream]/[ts_url][param];
    }
}


```

指定配置文件启动 srs
```bash
./objs/srs -c conf/srs.woniu.conf
```

### Openfire

-  4.4.4
```bash
docker run --name openfire -d --restart=always \
  --publish 9090:9090 --publish 5222:5222 --publish 7070:7070 \
  --volume /srv/docker/openfire:/var/lib/openfire \
  gizmotronic/openfire:4.4.4
```

### TURN Server

vi Dockerfile

```bash
# Coturn
#
# VERSION 4.4.2.2
FROM  ubuntu:14.04
MAINTAINER xzh<xzh@gmail.com>

RUN apt-get update && apt-get install -y \
  curl \
  libevent-core-2.0-5 \
  libevent-extra-2.0-5 \
  libevent-openssl-2.0-5 \
  libevent-pthreads-2.0-5 \
  libhiredis0.10 \
  libmysqlclient18 \
  libpq5 \
  telnet \
  wget

RUN wget http://turnserver.open-sys.org/downloads/v4.4.2.2/turnserver-4.4.2.2-debian-wheezy-ubuntu-mint-x86-64bits.tar.gz \
  && tar xzf turnserver-4.4.2.2-debian-wheezy-ubuntu-mint-x86-64bits.tar.gz \
  && dpkg -i coturn_4.4.2.2-1_amd64.deb

COPY ./turnserver.sh /turnserver.sh

ENV TURN_USERNAME kurento
ENV TURN_PASSWORD kurento
ENV REALM kurento.org
ENV NAT true

EXPOSE 3478 3478/udp

ENTRYPOINT ["/turnserver.sh"]
```

turnserver.sh
```bash
#!/bin/bash
set -e

if [ $NAT = "true" -a -z "$EXTERNAL_IP" ]; then

  # Try to get public IP
  PUBLIC_IP=$(curl http://169.254.169.254/latest/meta-data/public-ipv4) || echo "No public ip found on http://169.254.169.254/latest/meta-data/public-ipv4"
  if [ -z "$PUBLIC_IP" ]; then
    PUBLIC_IP=$(curl http://icanhazip.com) || exit 1
  fi

  # Try to get private IP
  PRIVATE_IP=$(ifconfig | awk '/inet addr/{print substr($2,6)}' | grep -v 127.0.0.1) || exit 1
  export EXTERNAL_IP="$PUBLIC_IP/$PRIVATE_IP"
  echo "Starting turn server with external IP: $EXTERNAL_IP"
fi

echo 'min-port=49152' > /etc/turnserver.conf
echo 'max-port=65535' >> /etc/turnserver.conf
echo 'fingerprint' >> /etc/turnserver.conf
echo 'lt-cred-mech' >> /etc/turnserver.conf
echo "realm=$REALM" >> /etc/turnserver.conf
echo 'log-file stdout' >> /etc/turnserver.conf
echo "user=$TURN_USERNAME:$TURN_PASSWORD" >> /etc/turnserver.conf
[ $NAT = "true" ] && echo "external-ip=$EXTERNAL_IP" >> /etc/turnserver.conf

exec /usr/bin/turnserver "$@"
```

```bash
chmod -R 777 /home/coturn
sudo docker build --tag coturn .
sudo docker run -p 3478:3478 -p 3478:3478/udp coturn
```

## 11. other

### Nginx-fancyindex

```bash
docker run -d \
  -p 8083:80 \
  -p 8084:443 \
  -e HTTP_AUTH="off" \
  -e HTTP_USERNAME="admin" \
  -e HTTP_PASSWD="admin" \
  -v /home/my-files:/app/public \
  --restart unless-stopped \
  --mount type=tmpfs,destination=/tmp \
  80x86/nginx-fancyindex
```

### Zentao

```bash
mkdir -p /data/zbox
docker run -d -p 8080:80 -p 3316:3306 -e USER="admin" -e PASSWD="admin" -e BIND_ADDRESS="false" -e SMTP_HOST="163.177.90.125 smtp.exmail.qq.com" -v /data/zbox/:/opt/zbox/ --name zentao-server idoop/zentao:latest 
```

- 8080 访问禅道外部端口号
- 3316 把容器3306数据库端口映射到主机3316端口
- USER 设置登录账号 admin
- PASSWD 设置登录密码 123456
- BIND_ADDRESS 设置为false


### Node-RED

```bash
sudo docker run -it -p 1880:1880 --name=nodered --restart=always --user=root --net=host -v /data/nodered:/data -e TZ=Asia/Shanghai nodered/node-red
```


### Eclipse Che

http://ip:8080

```bash
docker run -it -d --rm \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /mydata/che:/data \
eclipse/che start
```


### Theia

```bash
chown -R 1000:1000 /mydata/
docker run -it -d -p 3000:3000 -v "/mydata/theia:/home/project:cached" theiaide/theia
docker run -it -d -p 3000:3000 -v "/mydata/theia-java:/home/project:cached" theiaide/theia-java
docker run -it -d --init -p 3000:3000 -v "/mydata/theia-full:/home/project:cached" theiaide/theia-full
```