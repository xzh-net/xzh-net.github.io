# MySQL 5.7.29

## 1. 安装

### 1.1 单机

1. 上传解压

```bash
cd /opt/software/
mkdir -p mysql
tar xvf mysql-5.7.29-1.el7.x86_64.rpm-bundle.tar -C /opt/software/mysql
```

2. 安装

```bash
# 卸载版本
rpm -qa|grep mariadb
rpm -qa | grep -i mysql
rpm -e mysql-community-server-5.7.29-1.el7.x86_64 --nodeps

yum -y install libaio
cd /opt/software/mysql
rpm -ivh mysql-community-common-5.7.29-1.el7.x86_64.rpm mysql-community-libs-5.7.29-1.el7.x86_64.rpm mysql-community-client-5.7.29-1.el7.x86_64.rpm mysql-community-server-5.7.29-1.el7.x86_64.rpm
```

3. 修改配置

```bash
vim /etc/my.cnf

datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

3. 初始化

```bash
mysqld --initialize 					# 初始化mysql
chown mysql:mysql /var/lib/mysql -R 	# 更改所属组
cat /var/log/mysqld.log | grep password	# 初始密码
systemctl start mysqld.service      	# 启动mysql
```

4. 登录数据库

```bash
mysql -u root -p
set password = password('123456');
grant all privileges on *.* to 'root' @'%' identified by '123456';
flush privileges;
```

5. 启动服务

```bash
systemctl start mysqld
systemctl stop mysqld
systemctl status mysqld
systemctl enable  mysqld    # 设置自启动成功
ps aux | grep mysqld        # 查看是否已经
```

### 1.2 卸载

1. 停止服务

```bash
systemctl stop mysqld.service
```

2. 删除依赖

```bash
rpm -qa | grep -i mysql 
yum remove -y mysql-community-libs-5.7.29-1.el7.x86_64 mysql-community-common-5.7.29-1.el7.x86_64 mysql-community-client-5.7.29-1.el7.x86_64 mysql-community-server-5.7.29-1.el7.x86_64

rpm -qa | grep -i mysql
```

3. 删除配置

```bash
find / -name mysql

rm -rf /var/lib/mysql
rm -rf /var/lib/mysql/mysql
rm -rf /usr/share/mysql
rm -rf /etc/my.cnf
rm -rf /var/log/mysqld.log
```

## 2. 库操作

### 2.1 建库用户授权

```sql
CREATE DATABASE IF NOT EXISTS sonar DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
GRANT ALL ON sonar.* TO 'sonar'@'%' IDENTIFIED BY '123456';
GRANT ALL ON sonar.* TO 'sonar'@'localhost' IDENTIFIED BY '123456';
flush privileges;
```

修改密码
```sql
--方法1，密码实时更新；修改用户“test”的密码为“1122”
set password for test =password('1122');
--方法2，需要刷新；修改用户“test”的密码为“1234”
update  mysql.user set  password=password('1234')  where user='test'
--刷新
flush privileges;
```

### 2.2 表名敏感配置

```sql
show variables like '%lower_case_table_names%' --大小写敏感

0 --大小写敏感。（Unix，Linux默认） 创建的库表将原样保存在磁盘上
1 --大小写不敏感。（Windows默认） 创建的库表时，MySQL将所有的库表名转换成小写存储在磁盘上
2 --大小写不敏感（OS X默认） 创建的库表将原样保存在磁盘上。 但SQL语句将库表名转换成小写
```

docker安装时配置文件位置
```bash
# 进入容器安装编辑器
apt-get update
apt-get install -y vim
# 修改配置文件
vi /etc/mysql/mysql.conf.d/mysqld.cnf 
lower_case_table_names=1
```

rpm安装时配置文件位置
```bash
find / -name my.cnf
/etc/mysql/mysql.conf.d/mysqld.cnf
lower_case_table_names=1
service mysql restart
```

### 2.3 慢查询日志


```bash
set global slow_query_log = on      #临时开启慢查询日志
set global slow_query_log = off     #临时关闭
set long_query_time = 1             #临时设置查询临界点
set globle log_output = file        #设置慢查询存储的方式
show variables like '%quer%'        #开启状态和慢查询日志储存的位置

cat -n  /data/mysql/mysql-slow.log  #查看示例
```

### 2.4 备份恢复

```bash
mysqldump -uroot -proot --all-databases >/tmp/all.sql                               # 备份所库
mysqldump -uroot -proot --databases db1 db2 >/tmp/user.sql                          # 备份指定库
mysqldump -uroot -proot --databases db1 --tables a1 --where='id=1'  >/tmp/a1.sql    # 备份指定库指定表
mysqldump -uroot -proot --no-data --databases db1 >/tmp/db1.sql                     # 只导指定库的表结构
mysqldump --set-gtid-purged=OFF -h 127.0.0.1 -u root -p 123456 dbname --ignore-table=dbname.tb1 --ignore-table=dbname.tb2 > /tmp/all.sql   # 忽略表
mysql -uroot -proot -h 127.0.0.1 -P 3306 sonar</tmp/all.sql                         # 导入
```

### 2.5 binlog

```bash
vi /etc/my.cnf

server_id=1
log_bin=mysql-bin
binlog_format=ROW
```

登录mysql验证
```bash
mysql -uroot -p1q2w3e4r -e "show variables like 'log_bin%'";
```

## 3. 表操作

获取所有表结构信息
```sql
select table_name tableName, engine engine, table_comment tableComment, create_time createTime from
		information_schema.tables
		where table_schema = (select database())
		and table_name like concat('%tableName%')
		order by create_time desc
```

获取表名及备注sql
```sql
select table_name tableName, engine, table_comment tableComment, create_time createTime from information_schema.tables
        where table_schema = (select database()) and table_name = #{tableName}
```

获取指定表的字段名称主键等
```sql
select column_name columnName, data_type dataType, column_comment columnComment, column_key columnKey, extra from information_schema.columns
        where table_name = #{tableName} and table_schema = (select database()) order by ordinal_position
```

## 4. SQL

### 4.1 FUNCTION

```sql
CREATE FUNCTION `F_ACTUSER`(v_FLOWCID varchar(40)) RETURNS longtext CHARSET utf8
BEGIN
	DECLARE v_STR LONGTEXT DEFAULT '';
	DECLARE V_INDEX BIGINT DEFAULT 1;
	DECLARE done INT DEFAULT 0;
	DECLARE V_USERNAME VARCHAR(400);
	DECLARE PATH_CUR CURSOR FOR (SELECT DISTINCT B.USERNAME FROM TS_FLOW_PATH_COM A
                INNER JOIN VJSP_USERS B
                   ON B.USERID = A.TS_MK_USERID
                WHERE A.FLOWCID = v_FLOWCID
                  AND C.FLOWZT NOT IN (-1, 2, 3));
									
	DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
	
	OPEN PATH_CUR;
	pathLoop: LOOP
		FETCH PATH_CUR INTO V_USERNAME;
		IF done =1 THEN
			LEAVE pathLoop; 
		END IF; 
		IF V_INDEX =1 THEN
			SET v_STR = CONCAT(v_STR,V_USERNAME);
			SET V_INDEX =2;
		ELSE
			SET v_STR = CONCAT(v_STR,',',V_USERNAME);
		END IF;
	END LOOP pathLoop;
	CLOSE PATH_CUR;
	RETURN v_STR;
END
```

### 4.2 PROCEDURE

```sql
CREATE PROCEDURE `PROC_INIT_FLOW_SINGLE`(IN V_PARTNERID VARCHAR(40),
                                IN v_FLOWCID VARCHAR(40),
                                IN v_SPID VARCHAR(40),
                                IN v_SPYJ VARCHAR(2000),
                                IN v_ZXSX BIGINT,
                                IN v_USERID VARCHAR(40)
                                )
BEGIN
	DECLARE V_LYID VARCHAR(40);
	DECLARE done INT DEFAULT 0; 
	DECLARE LIST_CUR CURSOR FOR (SELECT TS_MK_PID FROM TS_FLOW_PATH_COM WHERE FLOWCID = V_FLOWCID AND PARTNERID = V_PARTNERID AND TS_MK_ZX_SX = v_ZXSX AND TS_MK_ZX_SX >= 0  AND TS_MK_PID = v_SPID);
	
	DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

	OPEN LIST_CUR;
		listLoop: LOOP
			FETCH LIST_CUR INTO V_LYID;
			IF done=1 THEN
				LEAVE listLoop; 
			END IF; 
			UPDATE TS_FLOW_PATH_COM SET TS_MK_SQ_DATE = NOW(), TS_SJTIME = -1, TS_MK_SQ_ZT = 2, TS_MK_SQ_YJ = v_SPYJ, TS_BTN_ID = -1, TS_BTN_NAME = V_SPYJ WHERE  FLOWCID = V_FLOWCID AND PARTNERID = V_PARTNERID AND TS_MK_PID = V_LYID;
			UPDATE TS_SYSTEM_DYWJ SET CKZT = 1 WHERE  FLOWCID = V_FLOWCID AND PARTNERID = V_PARTNERID AND PATHID = V_LYID;
		END LOOP listLoop;
	CLOSE LIST_CUR;
END
```

## 5. 集群

### 5.1 主从复制

#### 5.1.1 主库安装

##### 5.1.1.1 创建配置文件

创建目录
```bash
mkdir -p /opt/mysql/master/conf
```

创建my.cnf文件
```bash
vim /opt/mysql/master/conf/my.cnf
```

内容如下
```conf
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
collation-server = utf8_unicode_ci
init-connect='SET NAMES utf8'
character-set-server = utf8
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
symbolic-links=0
skip-character-set-client-handshake

binlog_format=ROW
binlog_rows_query_log_events=1
server_id = 1
log-bin= mysql-bin
gtid_mode=on
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log_recovery=ON
sync_master_info=1
binlog_checksum=CRC32

slave-parallel-type=LOGICAL_CLOCK
slave_parallel_workers=4
binlog_transaction_dependency_tracking=WRITESET_SESSION
transaction_write_set_extraction=XXHASH64

transaction-isolation=READ-COMMITTED
read-only=0
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
expire_logs_days=30

max_connections=3600

# Custom config should go here
!includedir /etc/mysql/conf.d/
```

备注：

```lua
skip-character-set-client-handshake：忽略应用程序想要设置的其他字符集
binlog_format：为设置binlog格式
binlog_rows_query_log_events：在row模式下开启该参数，将把sql语句打印到binlog日志里面，默认是0(off);
server_id：这个的值必需所有mysql实例都不重复
log-bin：binlog的名称
gtid_mode：开启GTID，用来代替classic的复制方法
enforce_gtid_consistency：开启gtid的一些安全限制，阻止不安全的语句执行
master_info_repository=table和relay_log_info_repository=table：master.info和relay.info保存在表中
relay_log_recovery：当slave从库宕机后，假如relay-log损坏了，导致一部分中继日志没有处理，则自动放弃所有未执行的relay-log，并且重新从master上获取日志，这样就保证了relay-log的完整性
sync_master_info：每个事务都会刷新master.info
binlog_checksum：默认为NONE， 表示在图1的箭头1 不生成checksum， 这样就可以兼容旧版本的mysql。
slave-parallel-type：DATABASE为默认值，基于库的并行复制方式；LOGICAL_CLOCK基于组提交的并行复制方式
slave_parallel_workers：设置多少个SQL Thread（coordinator线程）来进行并行复制
binlog_transaction_dependency_tracking：控制是否使用WRITESET策略，WRITESET_SESSION是在写集合的基础上增加约束,保证按照前后顺序执行
transaction_write_set_extraction：控制检测事务依赖关系时采用的HASH算法
transaction-isolation：事务隔离级别
read-only：是否只读，0为否，1为是
replicate-ignore-db：配置忽略同步的数据库
expire_logs_days：binlog日志过期时间，默认不过期
max_connections：mysql最大连接数
```

##### 5.1.1.2 启动主库

创建启动命令
```
vim /opt/mysql/master/start.sh
```

内容如下:
```bash
MYSQL_DIR=/opt/mysql/master
docker stop mysql-m
docker rm mysql-m
docker run -d \
    -p 3306:3306 \
    --name mysql-m \
    -v ${MYSQL_DIR}/conf/my.cnf:/etc/mysql/my.cnf \
    -v ${MYSQL_DIR}/data/mysql:/var/lib/mysql \
    -v ${MYSQL_DIR}/log:/opt/mysql/log \
    -e MYSQL_ROOT_PASSWORD=1q2w3e4r \
    mysql:5.7.25
```

##### 5.1.1.3 主库创建用于同步的账号

登录进主库
```bash
## 进去docker容器内
docker exec -it mysql-m bash
## 登录mysql数据库
mysql -uroot -p1q2w3e4r
```

创建backup账号
```sql
GRANT REPLICATION SLAVE ON *.* to 'backup'@'%' identified by '123456';
```

#### 5.1.2 从库安装

进入从库的服务器执行以下操作，建议是不同于主库的服务器，如果服务器相同需要修改3306端口为其他的值

##### 5.1.2.1 创建配置文件

创建目录
```bash
mkdir -p /opt/mysql/slave/conf
```

创建my.cnf文件
```
vim /opt/mysql/slave/conf/my.cnf
```

内容如下：

```conf
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
collation-server = utf8_unicode_ci
init-connect='SET NAMES utf8'
character-set-server = utf8
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
symbolic-links=0
skip-character-set-client-handshake

binlog_format=ROW
binlog_rows_query_log_events=1
server_id = 2
log-bin= mysql-bin
gtid_mode=on
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log_recovery=ON
sync_master_info=1
binlog_checksum=CRC32

slave-parallel-type=LOGICAL_CLOCK
slave_parallel_workers=4

binlog_transaction_dependency_tracking=WRITESET_SESSION
transaction_write_set_extraction=XXHASH64

transaction-isolation=READ-COMMITTED
read-only=1
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
expire_logs_days=30

max_connections=3600

# Custom config should go here
!includedir /etc/mysql/conf.d/
```

不同于主库的配置如下
- server_id：设为2
- read-only：设为只读

##### 5.1.2.2 启动从库

创建启动命令
```
vim /opt/mysql/slave/start.sh
```

内容如下:
```bash
MYSQL_DIR=/opt/mysql/slave
docker stop mysql-s
docker rm mysql-s
docker run -d \
    -p 3306:3306 \
    --name mysql-m \
    -v ${MYSQL_DIR}/conf/my.cnf:/etc/mysql/my.cnf \
    -v ${MYSQL_DIR}/data/mysql:/var/lib/mysql \
    -v ${MYSQL_DIR}/log:/opt/mysql/log \
    -e MYSQL_ROOT_PASSWORD=1q2w3e4r \
    mysql:5.7.25
```

##### 5.1.2.3 关联主库

登录进从库
```bash
## 进去docker容器内
docker exec -it mysql-s bash
## 登录mysql数据库
mysql -uroot -p1q2w3e4r
```

执行关联master语句
```sql
change master to master_host='192.168.28.130',master_port=3306,master_user='backup',master_password='123456',MASTER_AUTO_POSITION=1;
```

```lua
master_host：为主库ip
master_port：主库端口
master_user：主库用于同步的帐号
master_password：主库用于同步的帐号密码
master_auto_position：slave连接master将使用基于GTID的复制协议
```

##### 5.1.2.4 启动并查看slave
```bash
## 启动slave
start slave;
## 查看slave的状态
show slave status\G
```
- Slave_IO_Running和Slave_SQL_Running 都为Yes就代表配置成功了
- Seconds_Behind_Master：为主从延时(ms)

##### 5.1.2.5 创建从库的普通用户

read_only=1只读模式，可以限定普通用户进行数据修改的操作，但不会限定具有super权限的用户（如超级管理员root用户）的数据修改操作，所以需要另外创建普通账号来操作从库
```sql
GRANT select,insert,update,delete,create,drop,alter ON *.* to 'zlt'@'%' identified by '1q2w3e4r';
```

#### 5.1.3 主库查看同步信息

登录主库，查看binlog线程，执行以下语句查看正在执行的线程:
```bash
show processlist	# 查看binlog线程
show slave hosts	# 查看所有从库信息
```

### 5.2 主从切换

#### 5.2.1 对主库进行锁表

```bash
flush tables with read lock;
```

在flush tables with read lock成功获得锁之前，必须等待所有语句执行完成（包括SELECT）。所以如果有个慢查询在执行，或者一个打开的事务，或者其他进程拿着表锁，flush tables with read lock就会被阻塞，直到所有的锁被释放


#### 5.2.2 检查master同步状态

在主库执行
```sql
show processlist;
```
如果显示Master has sent all binlog to slave; waiting for more updates，则可以执行下一步了，否则需要等待

#### 5.2.3 检查slave同步状态

在从库执行
```sql
show processlist;
```
确保显示为 Slave has read all relay log; waiting for more updates

#### 5.2.4 提升slave为master

在从库执行以下语句

```sql
stop slave
reset master
reset slave all
```
reset slave all 命令会删除从库的 replication 参数，之后 show slave status\G 的信息返回为空。


创建同步用户，在从库执行
```sql
GRANT REPLICATION SLAVE ON *.* to 'backup'@'%' identified by '123456';
```

修改配置并重启
- 修改从库的my.cnf文件，将read-only的值改为0并重启mysql

#### 5.2.5 将原来master变为slave

修改配置并重启
- 修改主库的my.cnf文件，将read-only的值改为1并重启mysql

在主库执行以下语句
```sql
reset master;
reset slave;
change master to master_host='192.168.28.130',master_port=3306,master_user='backup',master_password='123456',MASTER_AUTO_POSITION=1;
start slave;
```

查看新slave的状态
```bash
## 在主库下执行
show slave status\G
```

Slave_IO_Running和Slave_SQL_Running都显示Yes就代表成功了


### 5.3 主主复制

#### 5.3.1 主库配置文件

```conf
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
collation-server = utf8_unicode_ci
init-connect='SET NAMES utf8'
character-set-server = utf8
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
symbolic-links=0
skip-character-set-client-handshake

binlog_format=ROW
binlog_rows_query_log_events=1
server_id = 1
log-bin= mysql-bin
gtid_mode=on
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log_recovery=ON
sync_master_info=1
binlog_checksum=CRC32

slave-parallel-type=LOGICAL_CLOCK
slave_parallel_workers=4

binlog_transaction_dependency_tracking=WRITESET_SESSION
transaction_write_set_extraction=XXHASH64

transaction-isolation=READ-COMMITTED
read-only=0
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
slave-skip-errors=all
expire_logs_days=30

max_connections=3600

auto_increment_offset = 1
auto_increment_increment = 2

# Custom config should go here
!includedir /etc/mysql/conf.d/
```

#### 5.3.2 第二主库配置文件

```conf
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
collation-server = utf8_unicode_ci
init-connect='SET NAMES utf8'
character-set-server = utf8
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
symbolic-links=0
skip-character-set-client-handshake

binlog_format=ROW
binlog_rows_query_log_events=1
server_id = 2
log-bin= mysql-bin
gtid_mode=on
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log_recovery=ON
sync_master_info=1
binlog_checksum=CRC32

slave-parallel-type=LOGICAL_CLOCK
slave_parallel_workers=4

binlog_transaction_dependency_tracking=WRITESET_SESSION
transaction_write_set_extraction=XXHASH64

transaction-isolation=READ-COMMITTED
read-only=0
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
slave-skip-errors=all
expire_logs_days=30

max_connections=3600

auto_increment_offset = 2
auto_increment_increment = 2

# Custom config should go here
!includedir /etc/mysql/conf.d/
```

#### 5.3.3 总结

主主配置与主从配置的区别在于以下2点：
- 两个库的read-only都为0，同为可写可读
- 新增auto_increment_offset和auto_increment_increment参数，并且两个库的auto_increment_offset值不相同
