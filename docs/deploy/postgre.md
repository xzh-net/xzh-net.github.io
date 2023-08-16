# PostgreSQL 12.4

- 下载地址：http://www.postgresql.org/ftp/source/

## 1. 安装

### 1.1 yum安装

```bash
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install -y postgresql12-server
sudo /usr/pgsql-12/bin/postgresql-12-setup initdb
sudo systemctl enable postgresql-12
sudo systemctl start postgresql-12
```

### 1.2 编译安装

#### 1.2.1 下载

```bash
wget https://ftp.postgresql.org/pub/source/v12.4/postgresql-12.4.tar.gz --no-check-certificate
```

#### 1.2.2 安装依赖

```bash
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all
yum makecache
yum update
yum install -y gcc readline-devel zlib-devel docbook-dtds docbook-style-xsl fop libxslt readline-devel.x86_64 zlib-devel.x86_64 libmemcached libpqxx
```

#### 1.2.3 解压编译

```bash
tar -zxvf postgresql-12.4.tar.gz -C /home/
cd /home
cd postgresql-12.4
./configure
make && make install
```

#### 1.2.4 创建用户组和目录授权

```bash
useradd postgres
passwd postgres
mkdir -p /data/pgdata/12/data         # 数据主目录
mkdir -p /data/pgdata/12/archive      # 归档恢复路径
mkdir -p /data/pg_backup              # 备份路径

chown postgres /usr/local/pgsql       # 应用授权
chown postgres /data/pgdata/12/data   # 数据授权 
chown postgres /data/pgdata/12/archive
chown postgres /data/pg_backup 
```

#### 1.2.5 系统参数优化

1. 内核参数

```bash
vi /etc/sysctl.conf
```

```conf
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
kernel.shmmni = 4096
kernel.sem = 50100 64128000 50100 1280
fs.file-max = 7672460
net.ipv4.ip_local_port_range = 9000 65000
net.core.rmem_default = 1048576
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
```

```bash
sysctl -p
```

2. 资源限制锁

```bash
vi /etc/security/limits.conf
```

```conf
* soft nofile 131072
* hard nofile 131072
* soft nproc 131072
* hard nproc 131072
* soft core unlimited
* hard core unlimited
* soft memlock 50000000
* hard memlock 50000000
```

#### 1.2.6 设置环境变量

```bash
su - postgres
vi .bash_profile

export PGHOME=/usr/local/pgsql
export PGDATA=/data/pgdata/12/data
PATH=$PATH:$HOME/bin:$PGHOME/bin

source .bash_profile
# su - postgres
# psql --version
```

#### 1.2.7 数据库初使化

```bash
su - postgres
initdb -D $PGDATA
```

#### 1.2.8 应用配置

```bash
cd /data/pgdata/12/data
vi postgresql.conf    # 配置PostgreSQL数据库服务器的相应的参数
max_connections = 2000
listen_addresses = '*'
```

```conf
vi pg_hba.conf       # 配置对数据库的访问权限，末尾添加
host    all             all             0.0.0.0/0        md5  # 所有地址访问
host    replication     all             0.0.0.0/0        md5  # 物理备份 -R
```

#### 1.2.9 设置开机自启动

```bash
su - root
cd /home/postgresql-12.4/contrib/start-scripts
#vi linux
prefix=/usr/local/pgsql
PGDATA="/data/pgdata/12/data"
```

将linux文件拷贝到/etc/init.d/目录下，并命名为postgresql
```bash
chmod a+x linux
su root
cp linux /etc/init.d/postgresql
cd /etc/init.d
chkconfig --add postgresql
chkconfig
```

#### 1.2.10 启动服务

```bash
pg_ctl -D /data/pgdata/12/data -l logfile start
pg_ctl -D /data/pgdata/12/data -l logfile stop
pg_ctl -D /data/pgdata/12/data -l logfile restart
pg_ctl -D /data/pgdata/12/data -l logfile status

service postgresql start
service postgresql stop
service postgresql restart
```

#### 1.2.11 测试

```bash
psql
# 修改用户默认密码
ALTER USER postgres WITH PASSWORD 'postgres'; 
/usr/local/pgsql/bin/createdb test
/usr/local/pgsql/bin/psql test
```

### 1.4 主备流复制

#### 1.4.1 主节点

1. 修改pg_hba.conf

```bash
host replication replica 0.0.0.0/0 md5  # 没有添加，否则开启注释
```

2. 添加流复制用户

```sql
create role replica with replication login password '123456';
alter user replica with password '123456';
```

3. 修改postgresql.conf

```bash
listen_addresses = '*'                     
port = 5432
max_connections = 1200
superuser_reserved_connections = 10
full_page_writes = on
wal_log_hints = off
max_wal_senders = 50
hot_standby = on
log_destination = 'csvlog'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S'
log_rotation_age = 1d
log_rotation_size = 10MB
log_statement = 'mod'
log_timezone = 'PRC'
timezone = 'PRC'
unix_socket_directories = '/tmp'
shared_buffers = 512MB
temp_buffers = 16MB
work_mem = 32MB
effective_cache_size = 2GB
maintenance_work_mem = 128MB
#max_stack_depth = 2MB
dynamic_shared_memory_type = posix
## PITR
full_page_writes = on
wal_buffers = 16MB
wal_writer_delay = 200ms
commit_delay = 0
commit_siblings = 5
wal_level = replica # 支持wal归档和复制
archive_mode = on
archive_command = 'test ! -f /data/pgdata/12/data/archivedir/%f && cp %p /data/pgdata/12/data/archivedir/%f'
archive_timeout = 60s     # 切换到一个新的wal段时间，定时归档间隔
max_wal_senders = 4       # 流复制连接个数
wal_keep_segments = 16    # 流复制保留的最多的xlog数目
```

#### 1.4.2 从节点

1. 清空数据

```bash
rm -rf /data/pgdata/12/data/*         # 数据主目录
rm -rf /data/pgdata/12/archive/*      # 归档恢复路径
```

2. 恢复数据和归档

```bash
pg_basebackup -D /data/pg_backup/ -Ft -Pv -U postgres -h 192.168.3.200 -p 5432 -R # 备份base和pg_wal
tar xf base.tar -C $PGDATA
tar xf base.tar -C /data/pgdata/12/archive
```

3. 修改standby.signal

```bash
vi standby.signal 
standby_mode = 'on'
```

4. 修改postgre.auto.conf

```bash
primary_conninfo = 'user=replication password=123456 host=192.168.3.200 port=5432 sslmode=disable sslcompression=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any'
```

5. 验证

```bash
# 启动从库以后再启动主库
select pg_wal_replay_resume(); 
select pid,state,client_addr,sync_priority,sync_state from pg_stat_replication; # 监控状态[主]
psql -c "\x" -c "SELECT * FROM pg_stat_wal_receiver;"     # 监控状态[从]
```


## 2. 命令

### 2.1 基本信息

```bash
psql -h localhost -p 5432 -U postgres -W #使用指定用户和IP端口登陆
\q              #退出psql命令行
\du             #查看角色属性
\l              #查看数据库列表
\l *template*   #查看包含template字符的数据库
\c test         #切换到test数据库
\d              #查看当前schema中所有的表
\d [schema.]table   #查看表的结构
```

```bash
show data_directory;            # 查看数据目录
show archive_mode;              # 是否开启归档
select * from pg_ls_logdir();   # 查看日志目录文件
select pg_switch_wal();         # 切换日志
select pg_walfile_name(pg_current_wal_insert_lsn());      # 查看当前日志
select * from pg_ls_waldir() order by modification desc;  # 查看日志最后修改时间
select pg_create_restore_point('xzh-before-delete0227');  # 创建还原点
select pid,state,client_addr,sync_priority,sync_state from pg_stat_replication; # 监控状态[主]
psql -c "\x" -c "SELECT * FROM pg_stat_wal_receiver;"     # 监控状态[从]

select pg_current_xlog_location();  # 查看当前日志文件lsn位置：
select pg_current_wal_lsn();
select pg_current_xlog_insert_location(); # 当前xlog buffer中的insert位置
select pg_xlogfile_name(lsn); # 查看某个lsn对应的日志名
select pg_walfile_name(lsn);
select pg_xlogfile_name_offset('lsn');  # 查看某个lsn在日志中的偏移量
select pg_walfile_name_offset('lsn');
select pg_xlog_location_diff('lsn','lsn');  # 查看两个lsn位置的差距
select pg_wal_lsn_diff('lsn','lsn');
select pg_last_xlog_receive_location(); # 查看备库接收到的lsn位置
select pg_last_wal_receive_lsn();
select pg_last_xlog_relay_location(); # 查看备库回放的lsn位置
select pg_last_xact_replay_timestamp();
select pg_relation_filepath('test'::regclass); # 查看表的数据文件路径
select pg_relation_filenode('test');
select 'test'::regclass::oid; # 查看表的oid
select pg_backend_pid();  # 查看当前会话pid
select gernate_series(1,8,2); # 生成序列
select gen_random_uuid(); # 生成uuid(pg13新特性)
select pg_reload_conf();  # 重载配置文件信息
select pg_postmaster_start_time();  # 查看数据库启动时间
select has_any_column_privilege(user,table,privilege);  # 查看用户表、列等权限信息
select has_any_column_privilege(table,privilege);
select has_column_privilege(user,table,column,privilege);
select has_table_privilege(user,table,privilege);
select txid_current_snapshot(); # 查看当前快照信息
select pg_rotate_logfile(); # 切换一个运行日志
select pg_xlog_replay_pause();  # 暂停、恢复回放进程
select pg_xlog_replay_resume();
select pg_export_snapshot();  # 导出一个快照
select pg_relation_size();  # 查看对象的大小信息
select pg_table_size();
select pg_total_relation_size();
pg_create/drop_physical_replication_slot(slotname); # 物理、逻辑复制槽
pg_create_logical_replication_slot(slotname,decodingname);
pg_logical_slot_get_changes();
```

### 2.2 建表语句

```bash
set timezone = 'Etc/UTC';
set timezone = 'Asia/Shanghai';
show timezone;

CREATE TABLE "public"."car" (
  "id" int8 NOT NULL,
  "car_no" varchar(15),
  "start_price" numeric(11,2),
  "view_num" int4,
  "on_status" int2,
  "on_time" timestamp(6),
  "register_date" date,
  "create_user_id" int8,
  "create_time" timestamptz(6),
  "is_deleted" int2 DEFAULT 0,
  PRIMARY KEY ("id")
);

COMMENT ON COLUMN "public"."car"."car_no" IS '车辆编号';
COMMENT ON COLUMN "public"."car"."start_price" IS '起拍价格';
COMMENT ON COLUMN "public"."car"."view_num" IS '查看数量';
COMMENT ON COLUMN "public"."car"."on_status" IS '上架状态（0：下架；1：上架）';
COMMENT ON COLUMN "public"."car"."on_time" IS '上架时间';
COMMENT ON COLUMN "public"."car"."register_date" IS '注册日期';
COMMENT ON COLUMN "public"."car"."create_user_id" IS '创建用户id';
COMMENT ON COLUMN "public"."car"."create_time" IS '创建时间';
COMMENT ON COLUMN "public"."car"."is_deleted" IS '删除标识（0：否；1：是）';
```

## 3. 库操作

### 3.1 用户授权

```bash
create user "sonar" with password '123456';
create database "sonardb" template template1 owner "sonar";
grant all privileges on database "sonardb" to "sonar";
flush privileges

drop user sonar;  # 删除用户
drop database if exists sonardb;  # 删除库
select pg_terminate_backend(pid) from pg_stat_activity where DATNAME='sonar'; # 库锁释放
```

无法删除正在连接中的数据库

```sql
UPDATE pg_database SET datallowconn = 'true' WHERE datname = 'ec_user';
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'ec_user';
```

### 3.2 备份恢复

#### 3.2.1 逻辑备份

```bash
# 单表导出sql语句，多表使用-t sys_user -t sys_menu
pg_dump -h localhost -U postgres -p 5432 -W oauth_center -t oauth_client_details --column-inserts > oauth_client_details.sql
psql -h localhost -U postgres < /home/postgres/oauth_client_details.sql oauth_center    # sql还原

pg_dump -h localhost -p 5432 -U postgres -d oauth_center -F t -f oauth_center.sql       # 导出copy语句
pg_restore -h localhost -U postgres -d oauth_center -v oauth_center.sql                 # 还原copy语句

pg_dump -h localhost -U postgres -F c -f /home/postgres/oauth_center.dump oauth_center  # 二进制备份
pg_restore -h localhost -U postgres -d  oauth_center  /home/postgres/oauth_center.dump  # 二进制还原

pg_dump --help
pg_restore --help
```

#### 3.2.2 物理备份

```bash
pg_basebackup -D /data/pg_backup/ -Ft -Pv -U postgres -h localhost -p 5432 -R # 备份base和pg_wal
cd /data/pg_backup/
rm -rf /data/pgdata/12/data/*      # 清空数据库
rm -rf /data/pgdata/12/archive/*   # 清空wal 
tar xf base.tar -C $PGDATA
tar xf pg_wal.tar -C /data/pgdata/12/archive/

# vi postgresql.auto.conf 
primary_conninfo = 'user=postgres password=123456 host=localhost port=5432 sslmode=disable sslcompression=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any'
restore_command = 'cp /data/pgdata/12/archive/%f %p'
recovery_target = 'immediate'
# touch /data/pgdata/12/data/recovery.signal
# 启动数据库后
service postgresql start
select pg_wal_replay_resume();  # 停止恢复
```

#### 3.2.3 PITR数据恢复

```bash
# 恢复到指定事务id
pg_waldump  0000000100000084000000EC
# vi postgresql.auto.conf
restore_command = 'cp /data/pgdata/12/archive/%f %p'
recovery_target_xid='501'
# 恢复到指定时间
recovery_target_time = '2019-04-02 13:16:49.007657+08'
# 恢复到指定还原点
recovery_target_name = 'xzh-before-delete0227'
# 启动数据库后
select pg_wal_replay_resume();  # 停止恢复
```


#### 3.2.4 定时备份

```bash
crontab -e
30 1 * * * sh /data/shell/bakup.sh  # 每天凌晨1点半执行
```

```bash
#!/bin/bash
cur_time=$(date '+%Y-%m-%d')
sevendays_time=$(date -d -10days '+%Y-%m-%d')

/usr/local/pgsql/12.4/bin/pg_dump -h localhost -U postgres -F c -f /opt/db/vjsp10010260_$cur_time.dump VJSP10010260
scp /opt/db/vjsp10010260_$cur_time.dump root@192.168.42.38:/mydata/db
rm -rf /opt/db/vjsp10010260_$sevendays_time.dump
echo "backup finished" his
```

### 3.3 归档日志

#### 3.3.1 自动清理

```bash
mkdir -p $PGDATA/archivedir/  # 创建归档目录
vi $PGDATA/pg_archive.sh      # 轮转脚本
test ! -f $PGDATA/archivedir/$1 && cp --preserve=timestamps $2 $PGDATA/archivedir/$1 ; find $PGDATA/archivedir/ -type f -mtime +7 -exec rm -f {} \;


# 修改归档配置
wal_level = replica 
archive_mode = on
archive_command = 'pg_archive.sh %f %p'
```

#### 3.3.2 手动清理

```bash
su - postgres
cd /usr/local/pgsql/bin
./pg_controldata /data/pgdata/12/data/          # 查找最后一个同步块
cd /data/pgdata/12/data/archivedir
pg_archivecleanup ./ 0000000100000084000000EC   # 清除同步块
```

### 4.4 AWR报告

```bash
cd /home/postgresql-12.4/contrib/
make && make install
su - postgres
cd $PGDATA
#vi postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 1000
pg_stat_statements.track = all

service postgresql start
psql \c
create extension pg_stat_statements;
```

## 4. 表操作

### 4.1 锁表

```sql
-- 执行中sql
SELECT pgsa.datname AS database_name
    , pgsa.usename AS user_name
    , pgsa.client_addr AS client_addr
    , pgsa.application_name AS application_name
    , pgsa.state AS state
 , pgsa.backend_start AS backend_start
 , pgsa.xact_start AS xact_start
 , extract(epoch FROM now() - pgsa.xact_start) AS xact_time, pgsa.query_start AS query_start
 , extract(epoch FROM now() - pgsa.query_start) AS query_time
 , pgsa.query AS query_sql
FROM pg_stat_activity pgsa
WHERE pgsa.state != 'idle'
 AND pgsa.state != 'idle in transaction'
 AND pgsa.state != 'idle in transaction (aborted)'
ORDER BY query_time DESC
LIMIT 5

-- 查找锁表的pid
select pid from pg_locks l join pg_class t on l.relation = t.oid where t.relkind = 'r' and t.relname = 'lockedtable';

-- 查找锁表的语句
select pid, state, usename, query, query_start from pg_stat_activity 
where pid in ( select pid from pg_locks l join pg_class t on l.relation = t.oid and t.relkind = 'r' where t.relname =  'lockedtable');

-- 查找所有活动的被锁的表
select pid, state, usename, query, query_start 
from pg_stat_activity 
where pid in (
  select pid from pg_locks l 
  join pg_class t on l.relation = t.oid 
  and t.relkind = 'r' 
);

-- 解锁
SELECT pg_cancel_backend(pid);

-- PID查询sql
SELECT
	procpid,
	START,
	now( ) - START AS lap,
	current_query 
FROM
	(
	SELECT
		backendid,
		pg_stat_get_backend_pid ( S.backendid ) AS procpid,
		pg_stat_get_backend_activity_start ( S.backendid ) AS START,
		pg_stat_get_backend_activity ( S.backendid ) AS current_query 
	FROM
		( SELECT pg_stat_get_backend_idset ( ) AS backendid ) AS S 
	) AS S 
WHERE
	current_query <> '' 
	AND procpid = 34171 
ORDER BY
	lap DESC;

-- kill 进程
SELECT pg_terminate_backend ( pid )
```

### 4.2 统计


```sql
-- 查询所有数据库大小
SELECT
	d.datname AS NAME,
	pg_catalog.pg_get_userbyid ( d.datdba ) AS OWNER,
CASE
		WHEN pg_catalog.has_database_privilege ( d.datname, 'CONNECT' ) THEN
		pg_catalog.pg_size_pretty ( pg_catalog.pg_database_size ( d.datname ) ) ELSE'No Access' 
END AS SIZE 
FROM
	pg_catalog.pg_database d 
ORDER BY
CASE
		WHEN pg_catalog.has_database_privilege ( d.datname, 'CONNECT' ) THEN
		pg_catalog.pg_database_size ( d.datname ) 
END DESC 
	LIMIT 20
 
-- 查询所有表大小
select relname, pg_size_pretty(pg_relation_size(relid)) as size from pg_stat_user_tables;
 
-- 查询所有表的总大小，包括其索引大小
select relname, pg_size_pretty(pg_total_relation_size(relid)) as size from pg_stat_user_tables;

-- 查询单个索引大小
select pg_size_pretty(pg_relation_size('index')) as size;

-- 查询单个表空间大小
select pg_size_pretty(pg_tablespace_size('pg_default')) as size;
 
-- 查询所有表空间大小
select spcname, pg_size_pretty(pg_tablespace_size(spcname)) as size from pg_tablespace;
-- 或
select spcname, pg_size_pretty(pg_tablespace_size(oid)) as size from pg_tablespace;

-- 查询所有表的行数
SELECT schemaname,relname,n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;
```

### 4.3 表注释

```sql
-- 查询所有表注释
SELECT tb.table_name, d.description 
FROM information_schema.tables tb
         JOIN pg_class c ON c.relname = tb.table_name
         LEFT JOIN pg_description d ON d.objoid = c.oid AND d.objsubid = '0'
WHERE tb.table_schema = 'public';

-- 查询所有列注释
SELECT col.table_name, col.column_name, col.ordinal_position AS o, d.description
FROM information_schema.columns col
         JOIN pg_class c ON c.relname = col.table_name
         LEFT JOIN pg_description d ON d.objoid = c.oid AND d.objsubid = col.ordinal_position
WHERE col.table_schema = 'public'
ORDER BY col.table_name, col.ordinal_position;

-- 查询所有没注释的表
SELECT tb.table_name, d.description
FROM information_schema.tables tb
         JOIN pg_class c ON c.relname = tb.table_name
         LEFT JOIN pg_description d ON d.objoid = c.oid AND d.objsubid = '0'
WHERE tb.table_schema = 'public' AND d.description IS NULL;

-- 查询所有没注释的列
SELECT col.table_name, col.column_name, col.ordinal_position AS o, d.description
FROM information_schema.columns col
         JOIN pg_class c ON c.relname = col.table_name
         LEFT JOIN pg_description d ON d.objoid = c.oid AND d.objsubid = col.ordinal_position
WHERE col.table_schema = 'public' AND description IS NULL
ORDER BY col.table_name, col.ordinal_position;

-- 获取指定表的结构
SELECT C
	.relname AS 表名,
	A.attname AS 列名,
	( CASE WHEN A.attnotnull = TRUE THEN TRUE ELSE FALSE END ) AS 是否非空,
	(
	CASE
			
			WHEN (
			SELECT COUNT
				( pg_constraint.* ) 
			FROM
				pg_constraint
				INNER JOIN pg_class ON pg_constraint.conrelid = pg_class.oid
				INNER JOIN pg_attribute ON pg_attribute.attrelid = pg_class.oid 
				AND pg_attribute.attnum = ANY ( pg_constraint.conkey )
				INNER JOIN pg_type ON pg_type.oid = pg_attribute.atttypid 
			WHERE
				pg_class.relname = C.relname 
				AND pg_constraint.contype = 'p' 
				AND pg_attribute.attname = A.attname 
				) > 0 THEN
			TRUE ELSE FALSE 
			END 
			) AS 是否是主键,
			concat_ws ( '', T.typname ) AS 字段类型,
			( CASE WHEN A.attlen > 0 THEN A.attlen ELSE A.atttypmod - 4 END ) AS 长度,
			d.description AS 备注 
		FROM
			pg_class C,
			pg_attribute A,
			pg_type T,
			pg_description d 
		WHERE
			C.relname = 'product' 
			AND A.attnum > 0 
			AND A.attrelid = C.oid 
			AND A.atttypid = T.oid 
			AND d.objoid = A.attrelid 
			AND d.objsubid = A.attnum 
		ORDER BY
		C.relname DESC,
	A.attnum ASC
```

### 4.4 SQL监控

pg_stat_statements模块提供一种方法追踪一个服务器所执行的所有 SQL 语句的执行统计信息

```lua
userid	oid	pg_authid.oid	执行该语句的用户的 OID
dbid	oid	pg_database.oid	在其中执行该语句的数据库的 OID
queryid	bigint	 	内部哈希码，从语句的解析树计算得来
query	text	 	语句的文本形式
calls	bigint	 	被执行的次数
total_time	double precision	 	在该语句中花费的总时间，以毫秒计
min_time	double precision	 	在该语句中花费的最小时间，以毫秒计
max_time	double precision	 	在该语句中花费的最大时间，以毫秒计
mean_time	double precision	 	在该语句中花费的平均时间，以毫秒计
stddev_time	double precision	 	在该语句中花费时间的总体标准偏差，以毫秒计
rows	bigint	 	该语句检索或影响的行总数
shared_blks_hit	bigint	 	该语句造成的共享块缓冲命中总数
shared_blks_read	bigint	 	该语句读取的共享块的总数
shared_blks_dirtied	bigint	 	该语句弄脏的共享块的总数
shared_blks_written	bigint	 	该语句写入的共享块的总数
local_blks_hit	bigint	 	该语句造成的本地块缓冲命中总数
local_blks_read	bigint	 	该语句读取的本地块的总数
local_blks_dirtied	bigint	 	该语句弄脏的本地块的总数
local_blks_written	bigint	 	该语句写入的本地块的总数
temp_blks_read	bigint	 	该语句读取的临时块的总数
temp_blks_written	bigint	 	该语句写入的临时块的总数
blk_read_time	double precision	 	该语句花在读取块上的总时间，以毫秒计（如果track_io_timing被启用，否则为零）
blk_write_time	double precision	 	该语句花在写入块上的总时间，以毫秒计（如果track_io_timing被启用，否则为零）
```

```sql
-- 查询单次调用最耗 IO SQL TOP 5
SELECT userid::regrole, dbid, query FROM pg_stat_statements ORDER BY (blk_read_time+blk_write_time)/calls DESC LIMIT 5;
-- 查询总最耗 IO SQL TOP 5
SELECT userid::regrole, dbid, query FROM pg_stat_statements ORDER BY (blk_read_time+blk_write_time) DESC LIMIT 5;

-- 查询单次调用最耗时 SQL TOP 5
SELECT userid::regrole, dbid, query FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 5;
-- 查询总最耗时 SQL TOP 5
SELECT userid::regrole, dbid, query FROM pg_stat_statements ORDER BY total_time DESC LIMIT 5;
-- 找不到mean_time字段的时候使用
SELECT
  userid AS 执行者ID,
  dbid AS 执行数据库ID,
  query AS 执行的语句 ,
  calls AS 执行次数 ,
  total_time AS 执行总时间,
  total_time / calls AS 执行平均时间,
  ROWS AS 影响的总行数 
FROM
  pg_stat_statements 
ORDER BY
  total_time / calls DESC 
  LIMIT 10

-- 响应时间抖动最严重 SQL
select userid::regrole, dbid, query from pg_stat_statements order by stddev_time desc limit 5;  
-- 最耗共享内存 SQL
select userid::regrole, dbid, query from pg_stat_statements order by (shared_blks_hit+shared_blks_dirtied) desc limit 5;
-- 最耗临时空间 SQL
select userid::regrole, dbid, query from pg_stat_statements order by temp_blks_written desc limit 5;  
-- 清理历史统计信息
select pg_stat_statements_reset(); 
```


## 5. PG/SQL

### 5.1 VIEW

1. dual解决方案

```sql
CREATE VIEW "public"."dual" AS  SELECT 'X'::character varying(1) AS dummy;
ALTER TABLE "public"."dual" OWNER TO "postgres";
```

2. 普通视图

```sql
CREATE VIEW "public"."view_product" AS  SELECT product.id,
    product.create_user_id,
    product.create_time
   FROM product;

ALTER TABLE "public"."view_product" OWNER TO "postgres";
```

3. 拼接视图

```sql
CREATE VIEW "view_of_user" AS  SELECT ofuser.username,
    ofuser.email,
    ofuser.plainpassword AS password,
    ofuser.username AS loginid
   FROM ofuser
UNION
 SELECT ts_sso_user_info.ts_sso_user_info_id AS username,
    ts_sso_user_info.ts_email AS email,
    ts_sso_user_info.ts_sso_user_info_id AS password,
    ts_sso_user_info.ts_sso_user_info_id AS loginid
   FROM ts_sso_user_info;

ALTER TABLE "view_of_user" OWNER TO "postgres";
```

### 5.2 TRIGGER

1. 分数表

```sql
CREATE TABLE stu_score
(
  stuno serial NOT NULL,       --学生编号
  major character varying(16), --专业课程
  score integer                --分数
)
WITH (
  OIDS=FALSE
);
ALTER TABLE stu_score OWNER TO postgres;
```

2. 汇总表

```sql
CREATE TABLE major_stats
(
  major character varying(16), --专业课程
  total_score integer,         --总分
  total_students integer       --学生总数
)
WITH (
  OIDS=FALSE
);
ALTER TABLE major_stats OWNER TO postgres;
```

3. 计算函数
   
```sql
create or replace function fun_stu_major()
returns trigger as 
	$BODY$
	DECLARE
	rec record;
	BEGIN
DELETE FROM major_stats;--将统计表里面的旧数据清空
FOR rec IN (SELECT major,sum(score) as total_score,count(*) as total_students 
FROM stu_score GROUP BY major) LOOP
INSERT INTO major_stats VALUES(rec.major,rec.total_score,rec.total_students);
END LOOP;
return NEW;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100;
```

4. 触发规则

```sql
create trigger tri_stu_major 
AFTER insert or update or delete
on stu_score 
for each row
execute procedure fun_stu_major()
```

### 5.3 FUNCTION

#### 5.3.1 循环函数

```sql
CREATE OR REPLACE FUNCTION "public"."f_actuser"("v_flowcid" text)
  RETURNS "pg_catalog"."text" AS $BODY$
DECLARE

  v_STR   text;
  V_INDEX bigint;
 item record;
BEGIN
  v_STR   := '';
  V_INDEX := 1;
  FOR ITEM IN (SELECT distinct B.USERNAME
                 FROM TS_FLOW_PATH_COM A
                INNER JOIN VJSP_USERS B
                   ON B.USERID = A.TS_MK_USERID
                INNER JOIN TS_FLOW_MAIN_MX C
                   ON C.FLOWCID = A.FLOWCID
                WHERE A.FLOWCID = v_FLOWCID
                  AND A.TS_MK_DEL = 0
                  AND A.TS_MK_SQ_ZT = 1
                  AND C.FLOWZT NOT IN (-1, 2, 3)) LOOP
    IF V_INDEX = 1 THEN
      v_STR := ITEM.USERNAME;
    ELSE
      v_STR := v_STR || ',' || ITEM.USERNAME;
    END IF;
    V_INDEX := V_INDEX + 1;
  END LOOP;
  RETURN v_STR;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100
```

#### 5.3.2 执行sql函数

```sql
CREATE OR REPLACE FUNCTION "public"."Untitled"("formno" text)
  RETURNS "pg_catalog"."varchar" AS $BODY$
DECLARE
  RETVAL             varchar(200);
  V_MOUDLE_ID        varchar(200);
  V_PREFIX           varchar(200);
  V_CONTAIN_YEAR     bigint;
  V_CONTAIN_MONTH    bigint;
  V_CONTAIN_DAY      bigint;
  V_NUMBER_LENGTH    int;
  V_NUMBER_BEGIN     varchar(200);
  V_MAX_MID          varchar(200);
  V_MAX_MNUM         bigint;
  V_MAX_MNUM_STR     varchar(200);
  V_LOCK_SQL         varchar(4000);
  V_FIND_MAX_NUM_SQL varchar(4000);
  V_SQL_HEAD         varchar(4000);
  V_SQL_BODY         varchar(4000);
  V_YEAR             varchar(40);
  V_MONTH            varchar(40);
  V_DAY              varchar(40);
BEGIN
   RETVAL:='';
   LOCK table VJSP_CRM_NUMBER_RULE_LOCK in SHARE mode;
   --获取当前人员角色ID
  SELECT MAX(A.MOUDLE_ID),
         MAX(A.PREFIX),
         MAX(A.CONTAIN_YEAR),
         MAX(A.CONTAIN_MONTH),
         MAX(A.CONTAIN_DAY),
         MAX(A.NUMBER_LENGTH),
         MAX(A.NUMBER_BEGIN)
    INTO V_MOUDLE_ID,
         V_PREFIX,
         V_CONTAIN_YEAR,
         V_CONTAIN_MONTH,
         V_CONTAIN_DAY,
         V_NUMBER_LENGTH,
         V_NUMBER_BEGIN
    FROM VJSP_CRM_NUMBER_RULE A
   WHERE A.MOUDLE_ID = FORMNO;
  IF COALESCE(V_MOUDLE_ID,'x'::text)='x' THEN
    SELECT MAX(A.MOUDLE_ID),
           MAX(A.PREFIX),
           MAX(A.CONTAIN_YEAR),
           MAX(A.CONTAIN_MONTH),
           MAX(A.CONTAIN_DAY),
           MAX(A.NUMBER_LENGTH),
           MAX(A.NUMBER_BEGIN)
      INTO V_MOUDLE_ID,
           V_PREFIX,
           V_CONTAIN_YEAR,
           V_CONTAIN_MONTH,
           V_CONTAIN_DAY,
           V_NUMBER_LENGTH,
           V_NUMBER_BEGIN
      FROM VJSP_CRM_NUMBER_RULE A
     WHERE A.IS_DEFAULT = 1;
    IF COALESCE (V_MOUDLE_ID,'x'::text)='x' THEN
      RETURN RETVAL;
    END IF;
  END IF;
  --前缀
  RETVAL             := RETVAL || V_PREFIX;
  V_FIND_MAX_NUM_SQL := 'SELECT MAX(A.MOUDLE_ID), MAX(A.MOUDLE_NUMBER) FROM VJSP_CRM_NUMBER_RULE_DETAIL A WHERE A.MOUDLE_ID =''' ||
                        V_MOUDLE_ID || '''';
  V_SQL_HEAD         := 'INSERT INTO VJSP_CRM_NUMBER_RULE_DETAIL (MOUDLE_ID, MOUDLE_NUMBER';
  V_SQL_BODY         := ' VALUES ($1, $2';
  IF V_CONTAIN_YEAR = 1 THEN
    V_YEAR             := TO_CHAR(LOCALTIMESTAMP, 'YYYY');
    RETVAL             := RETVAL || V_YEAR;
    V_FIND_MAX_NUM_SQL := V_FIND_MAX_NUM_SQL || ' AND A.M_YEAR=''' ||
                          V_YEAR || '''';
    V_SQL_HEAD         := V_SQL_HEAD || ',M_YEAR';
    V_SQL_BODY         := V_SQL_BODY || ' ,''' || V_YEAR || '''';
  END IF;
  IF V_CONTAIN_MONTH = 1 THEN
    V_MONTH            := TO_CHAR(LOCALTIMESTAMP, 'MM');
    RETVAL             := RETVAL || V_MONTH;
    V_FIND_MAX_NUM_SQL := V_FIND_MAX_NUM_SQL || ' AND A.M_MONTH=''' ||
                          V_MONTH || '''';
    V_SQL_HEAD         := V_SQL_HEAD || ',M_MONTH';
    V_SQL_BODY         := V_SQL_BODY || ' ,''' || V_MONTH || '''';
  END IF;
  IF V_CONTAIN_DAY = 1 THEN
    V_DAY              := TO_CHAR(LOCALTIMESTAMP, 'DD');
    RETVAL             := RETVAL || V_DAY;
    V_FIND_MAX_NUM_SQL := V_FIND_MAX_NUM_SQL || ' AND A.M_DAY=''' || V_DAY || '''';
    V_SQL_HEAD         := V_SQL_HEAD || ',M_DAY';
    V_SQL_BODY         := V_SQL_BODY || ' ,''' || V_DAY || '''';
  END IF;
  V_SQL_HEAD := V_SQL_HEAD || ')';
  V_SQL_BODY := V_SQL_BODY || ')';

  EXECUTE V_FIND_MAX_NUM_SQL INTO V_MAX_MID, V_MAX_MNUM;
  IF COALESCE (V_MAX_MID,'x'::text)!='x' THEN
    V_MAX_MNUM := V_MAX_MNUM + 1;
  ELSE
    V_MAX_MNUM := V_NUMBER_BEGIN;
  END IF;
  EXECUTE V_SQL_HEAD || V_SQL_BODY
    USING V_MOUDLE_ID, V_MAX_MNUM;

  IF LENGTH(to_char(V_MAX_MNUM)) < V_NUMBER_LENGTH THEN
    V_MAX_MNUM_STR:= lpad(V_MAX_MNUM::text, V_NUMBER_LENGTH, '0'::text);
  ELSE
    V_MAX_MNUM_STR := V_MAX_MNUM;
  END IF;
  RETVAL := RETVAL || V_MAX_MNUM_STR;
  RETURN RETVAL;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100;

ALTER FUNCTION "public"."Untitled"("""formno""" "pg_catalog"."text") OWNER TO "postgres";
```

### 5.4 PROCEDURE

#### 5.4.1 返回游标过程

```sql
CREATE OR REPLACE FUNCTION "public"."proc_init_flow_cando"(IN "v_partnerid" text, IN "v_flowcid" text, IN "v_pathid" text, OUT "v_out" refcursor)
  RETURNS "pg_catalog"."refcursor" AS $BODY$
DECLARE

		V_SPZT   smallint;
		V_FLOWZT smallint;
		V_USER   varchar(2000);
		V_YJ     varchar(4000);s

BEGIN
		SELECT MAX(FLOWZT)
		INTO   V_FLOWZT
		FROM   TS_FLOW_MAIN_MX
		WHERE  FLOWCID = V_FLOWCID
		AND    PARTNERID = V_PARTNERID;
		SELECT MAX(A.TS_MK_SQ_ZT), MAX(B.USERNAME), MAX(A.TS_MK_SQ_YJ)
		INTO   V_SPZT, V_USER, V_YJ
		FROM   TS_FLOW_PATH_COM A
		INNER  JOIN VJSP_USERS B
		ON     A.TS_MK_USERID = B.USERID
		WHERE  A.TS_MK_PID = V_PATHID
		AND    A.PARTNERID = V_PARTNERID;
		
		IF V_SPZT != 1 THEN
				IF V_FLOWZT = 3 THEN
						SELECT MAX(A.TS_MK_SQ_ZT), MAX(B.USERNAME), MAX(A.TS_MK_SQ_YJ)
						INTO   V_SPZT, V_USER, V_YJ
						FROM   TS_FLOW_PATH_COM A
						INNER  JOIN VJSP_USERS B
						ON     A.TS_MK_USERID = B.USERID
						WHERE  A.FLOWCID = V_FLOWCID
						AND    A.TS_MK_SQ_ZT = 3
						AND    A.PARTNERID = V_PARTNERID;
				ELSIF V_FLOWZT = 2 THEN
						V_SPZT := 2;
				ELSE
						V_SPZT := 1;
				END IF;
				OPEN V_OUT FOR
						SELECT V_SPZT AS ZT, V_USER AS VUSER, V_YJ AS YJ ;
		ELSE
				OPEN V_OUT FOR
						SELECT 1  WHERE 1 = 2;
		END IF;
END;

 
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100
```

#### 5.4.2 执行sql过程

```sql
CREATE OR REPLACE FUNCTION "public"."vjsp_delete_crm_target"("v_year" int8=0, "v_tstype" int8=0, "v_spid" text=NULL::text, "v_sptypeid" text=NULL::text)
  RETURNS "pg_catalog"."void" AS $BODY$
DECLARE
  V_SQL varchar(4000);
BEGIN
  V_SQL := 'DELETE FROM VJSP_CRM_SALE_TARGET WHERE TSYEAR=' || V_YEAR ||
           ' AND TSTYPE=' || V_TSTYPE;
  IF V_SPID IS NOT NULL THEN
    V_SQL := V_SQL || ' AND SPID=''' || V_SPID || '''';
  END IF;
  IF V_SPTYPEID IS NOT NULL THEN
    V_SQL := V_SQL || ' AND SPTYPEID=''' || V_SPTYPEID || '''';
  END IF;
  EXECUTE IMMEDIATE V_SQL;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100
```

#### 5.4.3 遍历过程

```sql
CREATE OR REPLACE FUNCTION "public"."vjsp_crm_insert_seqdetail"("v_sfaid" text, "v_seqid" text, "v_execdate" timestamp)
  RETURNS "pg_catalog"."void" AS $BODY$
DECLARE
  V_STARTDATE timestamp;
  V_ENDDATE   timestamp;
  V_LOOPCOUNT bigint;
  ITEM record;
BEGIN
  FOR ITEM IN (SELECT * FROM VJSP_CRM_SFA_EVENT E WHERE E.SFA_ID = V_SFAID AND E.DEL_FLAG=0) LOOP
    IF ITEM.EXEC_DATE_FLAG = 1 THEN
      --无日期，直接插入
      INSERT INTO VJSP_CRM_SFA_SEQUENCE_DETAIL
        (ID, CUSTOMER_ID, DOC_CODE, DOC_TITLE, START_DATE, END_DATE, SFA_ID, SEQUENCE_ID, SFAEVENT_ID, STATUS, EXEC_DATE, EXEC_REMARK, CREATE_TIME, CREATER_ID, CREATER_NAME, DEPT_ID, DEPT_NAME, GSNM, EXEC_ORDER)
        SELECT f_getnid(), S.CUSTOMER_ID, '', '', NULL, NULL, V_SFAID, V_SEQID, ITEM.ID, 0, NULL, ITEM.EVENT_REMARK, f_now(), S.CREATER_ID, S.CREATER_NAME, S.DEPT_ID, S.DEPT_NAME, S.GSNM, ITEM.EVENT_ORDER
          FROM VJSP_CRM_SFA_SEQUENCE S
         WHERE S.ID = V_SEQID;
    ELSIF ITEM.EXEC_DATE_FLAG = 2 THEN
      --绝对日期
      INSERT INTO VJSP_CRM_SFA_SEQUENCE_DETAIL
        (ID, CUSTOMER_ID, DOC_CODE, DOC_TITLE, START_DATE, END_DATE, SFA_ID, SEQUENCE_ID, SFAEVENT_ID, STATUS, EXEC_DATE, EXEC_REMARK, CREATE_TIME, CREATER_ID, CREATER_NAME, DEPT_ID, DEPT_NAME, GSNM, EXEC_ORDER)
        SELECT f_getnid(), S.CUSTOMER_ID, '', '', ITEM.BEGIN_DATE, ITEM.END_DATE, V_SFAID, V_SEQID, ITEM.ID, 0, NULL, ITEM.EVENT_REMARK, f_now(), S.CREATER_ID, S.CREATER_NAME, S.DEPT_ID, S.DEPT_NAME, S.GSNM, ITEM.EVENT_ORDER
          FROM VJSP_CRM_SFA_SEQUENCE S
         WHERE S.ID = V_SEQID;
    ELSIF ITEM.EXEC_DATE_FLAG = 3 THEN
      --相对日期
      V_STARTDATE := V_EXECDATE+((ITEM.AFTER_DATE+1)||' day')::interval;
      V_ENDDATE   := V_EXECDATE+((ITEM.AFTER_DATE+ITEM.CONINUED_DAY)||' day')::interval;
      INSERT INTO VJSP_CRM_SFA_SEQUENCE_DETAIL
        (ID, CUSTOMER_ID, DOC_CODE, DOC_TITLE, START_DATE, END_DATE, SFA_ID, SEQUENCE_ID, SFAEVENT_ID, STATUS, EXEC_DATE, EXEC_REMARK, CREATE_TIME, CREATER_ID, CREATER_NAME, DEPT_ID, DEPT_NAME, GSNM, EXEC_ORDER)
        SELECT f_getnid(), S.CUSTOMER_ID, '', '', V_STARTDATE, V_ENDDATE, V_SFAID, V_SEQID, ITEM.ID, 0, NULL, ITEM.EVENT_REMARK, f_now(), S.CREATER_ID, S.CREATER_NAME, S.DEPT_ID, S.DEPT_NAME, S.GSNM, ITEM.EVENT_ORDER
          FROM VJSP_CRM_SFA_SEQUENCE S
         WHERE S.ID = V_SEQID;
    ELSIF ITEM.EXEC_DATE_FLAG = 4 THEN
      --循环
      V_STARTDATE := V_EXECDATE+(ITEM.AFTER_DATE||' day')::interval;
      V_ENDDATE   := V_EXECDATE+((ITEM.AFTER_DATE+ITEM.CONINUED_DAY-1)||' day')::interval;
      V_LOOPCOUNT := ITEM.LOOP_COUNT+1;
      FOR I IN 1 .. V_LOOPCOUNT LOOP
        INSERT INTO VJSP_CRM_SFA_SEQUENCE_DETAIL
        (ID, CUSTOMER_ID, DOC_CODE, DOC_TITLE, START_DATE, END_DATE, SFA_ID, SEQUENCE_ID, SFAEVENT_ID, STATUS, EXEC_DATE, EXEC_REMARK, CREATE_TIME, CREATER_ID, CREATER_NAME, DEPT_ID, DEPT_NAME, GSNM, EXEC_ORDER)
        SELECT f_getnid(), S.CUSTOMER_ID, '', '', V_STARTDATE, V_ENDDATE, V_SFAID, V_SEQID, ITEM.ID, 0, NULL, ITEM.EVENT_REMARK, f_now(), S.CREATER_ID, S.CREATER_NAME, S.DEPT_ID, S.DEPT_NAME, S.GSNM, ITEM.EVENT_ORDER
          FROM VJSP_CRM_SFA_SEQUENCE S
         WHERE S.ID = V_SEQID;
         
        V_STARTDATE := V_ENDDATE+((ITEM.EVERY_DAY+1)||' day')::interval;
        V_ENDDATE   := V_STARTDATE+((ITEM.CONINUED_DAY-1)||' day')::interval;
      END LOOP;
    END IF;
  END LOOP;
END;
 $BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER
  COST 100
```