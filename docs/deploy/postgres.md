# PostgreSQL 12.4

- 下载地址：http://www.postgresql.org/ftp/source/

## 1. 安装

### 1.1 编译安装

#### 1.1.1 下载

```bash
wget https://ftp.postgresql.org/pub/source/v12.4/postgresql-12.4.tar.gz --no-check-certificate
```

#### 1.1.2 安装依赖

```bash
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all
yum makecache
yum update
yum install -y gcc readline-devel zlib-devel docbook-dtds docbook-style-xsl fop libxslt readline-devel.x86_64 zlib-devel.x86_64 libmemcached libpqxx
```

#### 1.1.3 解压编译

```bash
tar -zxvf postgresql-12.4.tar.gz -C /opt/software
cd /opt/software/postgresql-12.4
./configure
make && make install
```

#### 1.1.4 创建用户组和目录授权

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

#### 1.1.5 系统参数优化

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

#### 1.1.6 配置环境变量

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

#### 1.1.7 数据库初使化

```bash
su - postgres
initdb -D $PGDATA
```

#### 1.1.8 应用配置

1. 修改参数

```bash
cd /data/pgdata/12/data
vi postgresql.conf    
```

```conf
max_connections = 2000
listen_addresses = '*'
```

2. 配置访问权限

```bash
vi pg_hba.conf
```

```conf
# "local" is for Unix domain socket connections only
local   all             all                                  trust
# IPv4 local connections:
host    all             all             0.0.0.0/0            md5
# IPv6 local connections:
host    all             all             ::1/128              md5
# Allow replication connections
host    replication     all             0.0.0.0/0            md5
```

#### 1.1.9 设置开机自启动

```bash
su - root
cd /opt/postgresql-12.4/contrib/start-scripts
vi linux
# 找到对应位置修改内容
prefix=/usr/local/pgsql
PGDATA="/data/pgdata/12/data"
```

将linux文件拷贝到`/etc/init.d/`目录下，并命名为`postgresql`

```bash
chmod a+x linux
cp linux /etc/init.d/postgresql
cd /etc/init.d
chkconfig --add postgresql
chkconfig
```

#### 1.1.10 启动服务

```bash
pg_ctl -D /data/pgdata/12/data -l logfile start
pg_ctl -D /data/pgdata/12/data -l logfile stop
pg_ctl -D /data/pgdata/12/data -l logfile restart
pg_ctl -D /data/pgdata/12/data -l logfile status

service postgresql start
service postgresql stop
service postgresql restart
```

#### 1.1.11 测试

```bash
/usr/local/pgsql/bin/createdb test
/usr/local/pgsql/bin/psql test

# 修改用户默认密码
psql
ALTER USER postgres WITH PASSWORD 'postgres'; 
```

```bash
psql -h localhost -p 5432 -U postgres -W # 使用指定用户和IP端口登陆
\q                  # 退出psql命令行
\du                 # 查看角色属性
\l                  # 查看数据库列表
\l *template*       # 查看包含template字符的数据库
\c test             # 切换到test数据库
\d                  # 查看当前schema中所有的表
\d [schema.]table   #查看表的结构
```

### 1.2 主备流复制

#### 1.2.1 主节点

1. 添加流复制用户

```sql
create role replica with replication login password '123456';
```

2. 设置流复制权限

```bash
cd /data/pgdata/12/data
vi pg_hba.conf
```

```conf
host replication replica 0.0.0.0/0 md5      # 没有添加，否则开启注释
```

3. 修改配置文件

```bash
vi postgresql.conf
```

```conf
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
log_filename = 'postgresql-%d.log'      # 保存一个月的日志,每天一个文件
log_truncate_on_rotation = on
log_file_mode = 0600
log_rotation_age = 1d
log_rotation_size = 0
log_statement = 'all'
log_timezone = 'Asia/Shanghai'
timezone = 'Asia/Shanghai'
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
wal_level = replica         # 开启归档
archive_mode = on
archive_command = 'test ! -f /data/pgdata/12/archive/%f && cp %p /data/pgdata/12/archive/%f'
archive_timeout = 60s       # 切换到一个新的wal段时间，定时归档间隔
max_wal_senders = 4         # 流复制连接个数
wal_keep_segments = 16      # 流复制保留的最多的xlog数目
```

4. 启动主库

```bash
pg_ctl -D /data/pgdata/12/data -l logfile start
```

#### 1.2.2 从节点

1. 清空数据

```bash
su - postgres
rm -rf /data/pgdata/12/data/*         # 数据主目录
rm -rf /data/pgdata/12/archive/*      # 归档恢复路径
```

2. 恢复数据和归档

```bash
pg_basebackup -D /data/pg_backup/ -Ft -Pv -U replica -h 192.168.3.200 -p 5432 -R    # 备份base和pg_wal
tar xf base.tar -C /data/pgdata/12/data
tar xf pg_wal.tar -C /data/pgdata/12/archive
```

3. 修改standby.signal

```bash
vi standby.signal 
standby_mode = 'on'
```

4. 修改配置

```bash
cd /data/pgdata/12/data
vi postgresql.auto.conf
```

```conf
primary_conninfo = 'user=replication password=123456 host=192.168.3.200 port=5432 sslmode=disable sslcompression=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any'
```

5. 启动从库

```bash
pg_ctl -D /data/pgdata/12/data -l logfile start
```


#### 1.2.3 验证主从

```bash
psql -c "\x" -c "select * from pg_stat_replication;"      # 监控状态[主]
psql -c "\x" -c "select * from pg_stat_wal_receiver;"     # 监控状态[从]
```

## 2. 模块

### 2.1 pg_stat_statements

pg_stat_statements模块提供一种方法追踪一个服务器所执行的所有SQL语句的执行统计信息

1. 编译插件

```bash
cd /opt/postgresql-12.4/contrib/pg_stat_statements
make && make install
su - postgres
cd $PGDATA
```

2. 修改配置

```bash
#vi postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
track_io_timing = on                # 用于跟踪IO消耗的时间
track_activity_query_size = 2048    # 设置单条SQL的最长长度，超过被截断显示（可选）
pg_stat_statements.max = 1000       # 采样参数，在pg_stat_statements中最多保留多少条统计信息，通过LRU算法，覆盖老的记录
pg_stat_statements.track = all      # all - (所有SQL包括函数内嵌套的SQL), top - 直接执行的SQL(函数内的sql不被跟踪), none - (不跟踪)
pg_stat_statements.track_utility = off  # 是否跟踪非DML语句 (例如DDL，DCL)，on表示跟踪, off表示不跟踪 
pg_stat_statements.save = on            # 重启后是否保留统计信息
```

3. 重启服务

```bash
pg_ctl -D /data/pgdata/12/data -l logfile restart
psql \c
create extension pg_stat_statements;
```

### 2.2 passwordcheck

passwordcheck模块是在`CREATE ROLE`或者`CREATE USER`期间检查用户密码是否符合指定的规则模块

1. 编译插件

```bash
cd /opt/postgresql-12.4/contrib/passwordcheck
make && make install
su - postgres
cd $PGDATA
```

2. 修改配置

```bash
#vi postgresql.conf
shared_preload_libraries = 'pg_stat_statements,passwordcheck'
```

3. 重启服务

```bash
pg_ctl -D /data/pgdata/12/data -l logfile restart
```

## 3. 库操作

### 3.1 用户管理

1. 创建用户

```sql
create user "sonar" with password '123456';
create database "sonardb" template template1 owner "sonar";
grant all privileges on database "sonardb" to "sonar";
```

2. 删除用户

```sql
drop user sonar;
drop database sonardb;    -- 删除库
select pg_terminate_backend(pid) from pg_stat_activity where DATNAME='sonar';    -- 库锁释放

-- 无法删除正在连接中的数据库
UPDATE pg_database SET datallowconn = 'true' WHERE datname = 'ec_user';
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'ec_user';
```

3. 查询用户

```sql
select * from pg_user;
select * from pg_database;
```


### 3.2 审计日志

1. 开启审计

```bash
cd /data/pgdata/12/data
vi postgresql.conf 
```

```conf
log_destination = 'csvlog'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%d.log'      # 保存一个月的日志,每天一个文件
log_truncate_on_rotation = on
log_file_mode = 0600
log_rotation_age = 1d
log_rotation_size = 0
log_statement = 'all'
log_timezone = 'Asia/Shanghai'
```

2. 重启应用

```bash
pg_ctl -D /data/pgdata/12/data -l logfile restart
```


### 3.3 归档日志

1. 开启归档

```bash
cd /data/pgdata/12/data
vi postgresql.conf 
```

```conf
wal_level = replica         # 开启归档
archive_mode = on
fsync = on
archive_command = 'test ! -f /data/pgdata/12/archive/%f && cp %p /data/pgdata/12/archive/%f'
archive_timeout = 60s
```

2. 重启应用

```bash
pg_ctl -D /data/pgdata/12/data -l logfile restart
```


3. 手动清理

```bash
su - postgres
pg_controldata -D /data/pgdata/12/data/ | grep 'REDO WAL file'
cd /data/pgdata/12/pg_wal
pg_archivecleanup ./ 0000000100000000000000C8   # 清除同步块
```

4. 相关命令

```sql
show data_directory;                -- 查看数据目录
show archive_mode;                  -- 是否开启归档
select * from pg_ls_logdir();       -- 查看日志目录文件
select pg_switch_wal();             -- 切换日志
select pg_walfile_name(pg_current_wal_insert_lsn());        -- 查看当前日志
select * from pg_ls_waldir() order by modification desc;    -- 查看日志最后修改时间
```


### 3.4 数据备份

#### 3.4.1 逻辑备份

1. pg_dump 

导出为SQL文件（明文格式），排除权限和所有者信息，只导出表结构及数据
```bash
pg_dump -h 127.0.0.1 -U postgres -p 5432  --no-owner --no-privileges oauth_center -f oauth_center.sql
```

导出为包含列名的插入语句，保证数据的一致性和完整性
```bash
# 导出多表使用-t tb1 -t tb2
pg_dump -h localhost -d oauth_center -U postgres -p 5432 -t oauth_center --column-inserts -f oauth_center.sql    
psql -h localhost -d oauth_center -U postgres -p 5432 -f oauth_center.sql
```

导出为tar格式的备份文件（二进制格式）
```bash
pg_dump -h localhost -U postgres -d oauth_center -p 5432 -Ft -f oauth_center.tar
pg_restore -h localhost -U postgres -d oauth_center -p 5432 -v oauth_center.tar
```

导出为自定义格式（支持压缩和并行恢复）
```bash
pg_dump -h localhost -U postgres -d oauth_center -p 5432 -Fc -f oauth_center.dmp
pg_restore -h localhost -U postgres -d oauth_center -p 5432 -v oauth_center.dmp
```

> pg_dump 详细参数

```lua
General options:
  -f, --file=FILENAME          output file or directory name
  -F, --format=c|d|t|p         output file format (custom, directory, tar,
                               plain text (default))
  -j, --jobs=NUM               use this many parallel jobs to dump
  -v, --verbose                verbose mode
  -V, --version                output version information, then exit
  -Z, --compress=0-9           compression level for compressed formats
  --lock-wait-timeout=TIMEOUT  fail after waiting TIMEOUT for a table lock
  --no-sync                    do not wait for changes to be written safely to disk
  -?, --help                   show this help, then exit

Options controlling the output content:
  -a, --data-only              dump only the data, not the schema
  -b, --blobs                  include large objects in dump
  -B, --no-blobs               exclude large objects in dump
  -c, --clean                  clean (drop) database objects before recreating
  -C, --create                 include commands to create database in dump
  -E, --encoding=ENCODING      dump the data in encoding ENCODING
  -n, --schema=PATTERN         dump the specified schema(s) only
  -N, --exclude-schema=PATTERN do NOT dump the specified schema(s)
  -O, --no-owner               skip restoration of object ownership in
                               plain-text format
  -s, --schema-only            dump only the schema, no data
  -S, --superuser=NAME         superuser user name to use in plain-text format
  -t, --table=PATTERN          dump the specified table(s) only
  -T, --exclude-table=PATTERN  do NOT dump the specified table(s)
  -x, --no-privileges          do not dump privileges (grant/revoke)
  --binary-upgrade             for use by upgrade utilities only
  --column-inserts             dump data as INSERT commands with column names
  --disable-dollar-quoting     disable dollar quoting, use SQL standard quoting
  --disable-triggers           disable triggers during data-only restore
  --enable-row-security        enable row security (dump only content user has
                               access to)
  --exclude-table-data=PATTERN do NOT dump data for the specified table(s)
  --extra-float-digits=NUM     override default setting for extra_float_digits
  --if-exists                  use IF EXISTS when dropping objects
  --inserts                    dump data as INSERT commands, rather than COPY
  --load-via-partition-root    load partitions via the root table
  --no-comments                do not dump comments
  --no-publications            do not dump publications
  --no-security-labels         do not dump security label assignments
  --no-subscriptions           do not dump subscriptions
  --no-synchronized-snapshots  do not use synchronized snapshots in parallel jobs
  --no-tablespaces             do not dump tablespace assignments
  --no-unlogged-table-data     do not dump unlogged table data
  --on-conflict-do-nothing     add ON CONFLICT DO NOTHING to INSERT commands
  --quote-all-identifiers      quote all identifiers, even if not key words
  --rows-per-insert=NROWS      number of rows per INSERT; implies --inserts
  --section=SECTION            dump named section (pre-data, data, or post-data)
  --serializable-deferrable    wait until the dump can run without anomalies
  --snapshot=SNAPSHOT          use given snapshot for the dump
  --strict-names               require table and/or schema include patterns to
                               match at least one entity each
  --use-set-session-authorization
                               use SET SESSION AUTHORIZATION commands instead of
                               ALTER OWNER commands to set ownership

Connection options:
  -d, --dbname=DBNAME      database to dump
  -h, --host=HOSTNAME      database server host or socket directory
  -p, --port=PORT          database server port number
  -U, --username=NAME      connect as specified database user
  -w, --no-password        never prompt for password
  -W, --password           force password prompt (should happen automatically)
  --role=ROLENAME          do SET ROLE before dump
```

2. pg_dumpall

导出所有数据库 + 全局对象（角色、表空间等），生成SQL脚本，全集群备份（适合小规模环境），但可能锁住全局对象，影响性能。不支持导出SQL文件以外的其他格式，如CSV、JSON等（-v或--verbose选项可以开启详细模式）

```bash
pg_dumpall -h localhost -U postgres -p 5432 -v -f all.sql
psql -h localhost -U postgres -p 5432 -f all.sql
```

> pg_dumpall 详细参数

```lua
General options:
  -f, --file=FILENAME          output file name
  -v, --verbose                verbose mode
  -V, --version                output version information, then exit
  --lock-wait-timeout=TIMEOUT  fail after waiting TIMEOUT for a table lock
  -?, --help                   show this help, then exit

Options controlling the output content:
  -a, --data-only              dump only the data, not the schema
  -c, --clean                  clean (drop) databases before recreating
  -E, --encoding=ENCODING      dump the data in encoding ENCODING
  -g, --globals-only           dump only global objects, no databases
  -O, --no-owner               skip restoration of object ownership
  -r, --roles-only             dump only roles, no databases or tablespaces
  -s, --schema-only            dump only the schema, no data
  -S, --superuser=NAME         superuser user name to use in the dump
  -t, --tablespaces-only       dump only tablespaces, no databases or roles
  -x, --no-privileges          do not dump privileges (grant/revoke)
  --binary-upgrade             for use by upgrade utilities only
  --column-inserts             dump data as INSERT commands with column names
  --disable-dollar-quoting     disable dollar quoting, use SQL standard quoting
  --disable-triggers           disable triggers during data-only restore
  --exclude-database=PATTERN   exclude databases whose name matches PATTERN
  --extra-float-digits=NUM     override default setting for extra_float_digits
  --if-exists                  use IF EXISTS when dropping objects
  --inserts                    dump data as INSERT commands, rather than COPY
  --load-via-partition-root    load partitions via the root table
  --no-comments                do not dump comments
  --no-publications            do not dump publications
  --no-role-passwords          do not dump passwords for roles
  --no-security-labels         do not dump security label assignments
  --no-subscriptions           do not dump subscriptions
  --no-sync                    do not wait for changes to be written safely to disk
  --no-tablespaces             do not dump tablespace assignments
  --no-unlogged-table-data     do not dump unlogged table data
  --on-conflict-do-nothing     add ON CONFLICT DO NOTHING to INSERT commands
  --quote-all-identifiers      quote all identifiers, even if not key words
  --rows-per-insert=NROWS      number of rows per INSERT; implies --inserts
  --use-set-session-authorization
                               use SET SESSION AUTHORIZATION commands instead of
                               ALTER OWNER commands to set ownership

Connection options:
  -d, --dbname=CONNSTR     connect using connection string
  -h, --host=HOSTNAME      database server host or socket directory
  -l, --database=DBNAME    alternative default database
  -p, --port=PORT          database server port number
  -U, --username=NAME      connect as specified database user
  -w, --no-password        never prompt for password
  -W, --password           force password prompt (should happen automatically)
  --role=ROLENAME          do SET ROLE before dump
```

#### 3.4.2 物理备份

1. 备份base和pg_wal

```bash
pg_basebackup -D /data/pg_backup/ -Ft -Pv -U postgres -h localhost -p 5432 -R
```

2. 清空数据

```bash
rm -rf /data/pgdata/12/data/*      # 清空数据库
rm -rf /data/pgdata/12/archive/*   # 清空wal 
```

3. 还原数据

```bash
cd /data/pg_backup/
tar xf base.tar -C $PGDATA
tar xf pg_wal.tar -C /data/pgdata/12/archive/
```

4. 修改配置

```bash
# vi postgresql.auto.conf 
primary_conninfo = 'user=postgres password=postgres host=localhost port=5432 sslmode=disable sslcompression=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any'
restore_command = 'cp /data/pgdata/12/archive/%f %p'
recovery_target = 'immediate'
```

```bash
touch /data/pgdata/12/data/recovery.signal
```

5. 启动数据库

```bash
pg_ctl -D /data/pgdata/12/data -l logfile start
select pg_wal_replay_resume();  # 停止恢复
```

#### 3.4.3 PITR数据恢复

```sql
select pg_create_restore_point('point-20231206');  -- 创建还原点
```


1. 修改配置

```bash
# 恢复到指定事务id
pg_waldump  0000000100000084000000EC
# vi postgresql.auto.conf
restore_command = 'cp /data/pgdata/12/archive/%f %p'
recovery_target_xid='501'
# 恢复到指定时间
recovery_target_time = '2019-04-02 13:16:49.007657+08'
# 恢复到指定还原点
recovery_target_name = 'point-20231206'
```

2. 启动数据库

```bash
pg_ctl -D /data/pgdata/12/data -l logfile start
select pg_wal_replay_resume();  # 停止恢复
```

#### 3.4.4 定时备份

```bash
crontab -e
30 1 * * * sh /data/shell/bakup.sh  # 每天凌晨1点半执行
```

```bash
#!/bin/bash
cur_time=$(date '+%Y-%m-%d')
find /data/pg_backup -mtime +30 -type f -name '*.tgz' -exec rm {} \;
## 备份
/usr/local/pgsql/12.4/bin/pg_dump -h localhost -U postgres -F c -f /data/pg_backup/user_center.$cur_time.dmp user_center
/usr/local/pgsql/12.4/bin/pg_dump -h localhost -U postgres -F c -f /data/pg_backup/oauth_center.$cur_time.dmp oauth_center
## 打包
tar zcvf pgsql-backup.$cur_time.tgz *.dmp
## 传输（2选1）
scp pgsql-backup.$cur_time.tgz postgres@192.168.2.100:/data/pg_backup
sshpass -p "123456" scp -o StrictHostKeyChecking=no pgsql-backup.$cur_time.tgz postgres@192.168.2.100:/data/pg_backup
## 删除备份
rm -rf /data/pg_backup/*.dmp
echo "backup finished" his
```

### 3.5 表空间管理

```sql
-- 查询单个表空间大小
select pg_size_pretty(pg_tablespace_size('pg_default')) as size;
 
-- 查询所有表空间大小
select spcname, pg_size_pretty(pg_tablespace_size(spcname)) as size from pg_tablespace;
-- 或
select spcname, pg_size_pretty(pg_tablespace_size(oid)) as size from pg_tablespace;
```

## 4. 表操作

### 4.1 查询表结构

```sql
-- 查询所有数据库大小
SELECT
    d.datname AS NAME,
    pg_catalog.pg_get_userbyid ( d.datdba ) AS OWNER,
CASE
        WHEN pg_catalog.has_database_privilege ( d.datname, 'CONNECT' ) THEN
        pg_catalog.pg_size_pretty ( pg_catalog.pg_database_size ( d.datname ) ) ELSE 'No Access' 
    END AS SIZE 
FROM
    pg_catalog.pg_database d 
ORDER BY
CASE
        WHEN pg_catalog.has_database_privilege ( d.datname, 'CONNECT' ) THEN
        pg_catalog.pg_database_size ( d.datname ) 
END DESC 
    LIMIT 20

-- 获取指定表的结构
SELECT
    C.relname AS 表名,
    A.attname AS 列名,
    ( CASE WHEN A.attnotnull = TRUE THEN TRUE ELSE FALSE END ) AS 是否非空,
    (
    CASE
            
            WHEN (
            SELECT
                COUNT( pg_constraint.* ) 
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
            C.relname = 'tableName' 
            AND A.attnum > 0 
            AND A.attrelid = C.oid 
            AND A.atttypid = T.oid 
            AND d.objoid = A.attrelid 
            AND d.objsubid = A.attnum 
        ORDER BY
        C.relname DESC,
    A.attnum ASC

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

```

### 4.2 创建表

```sql
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

### 4.3 锁表

```sql
-- 查询当前正在执行所有SQL语句
SELECT
    pid,
    datname,
    usename,
    client_addr,
    application_name,
    STATE,
    backend_start,
    xact_start,
    xact_stay,
    query_start,
    query_stay,
    REPLACE ( query, chr( 10 ), ' ' ) AS query 
FROM
    (
    SELECT
        pgsa.pid AS pid,
        pgsa.datname AS datname,
        pgsa.usename AS usename,
        pgsa.client_addr client_addr,
        pgsa.application_name AS application_name,
        pgsa.STATE AS STATE,
        pgsa.backend_start AS backend_start,
        pgsa.xact_start AS xact_start,
        EXTRACT ( epoch FROM ( now( ) - pgsa.xact_start ) ) AS xact_stay,
        pgsa.query_start AS query_start,
        EXTRACT ( epoch FROM ( now( ) - pgsa.query_start ) ) AS query_stay,
        pgsa.query AS query 
    FROM
        pg_stat_activity AS pgsa 
    WHERE
        pgsa.STATE != 'idle' 
        AND pgsa.STATE != 'idle in transaction' 
        AND pgsa.STATE != 'idle in transaction (aborted)' 
    ) idleconnections 
ORDER BY
    query_stay DESC

-- 根据pid查询sql
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

-- kill 执行时间大于90s的sql
SELECT 'SELECT pg_terminate_backend(' || pid || ');'
FROM pg_stat_activity pgsa
WHERE  pid != pg_backend_pid()
  and pgsa.STATE != 'idle' 
        AND pgsa.STATE != 'idle in transaction' 
        AND pgsa.STATE != 'idle in transaction (aborted)' 
        AND EXTRACT ( epoch FROM ( now( ) - pgsa.query_start ) ) > 90

-- kill 进程
SELECT pg_terminate_backend ( pid );    --彻底停止进程，导致连接关闭。事务会回滚，释放持有的锁
-- 解锁
SELECT pg_cancel_backend(pid);          --只是中断正在运行的查询，连接仍然存在

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
```

### 4.4 创建视图

#### 4.4.1 虚拟视图

`public`模式下创建一个模仿Oracle数据库中的DUAL表

```sql
CREATE VIEW "public"."dual" AS  SELECT 'X'::character varying(1) AS dummy;
ALTER TABLE "public"."dual" OWNER TO "postgres";
```

#### 4.4.2 普通视图

```sql
CREATE VIEW "public"."view_product" AS  SELECT product.id,
    product.create_user_id,
    product.create_time
   FROM product;

ALTER TABLE "public"."view_product" OWNER TO "postgres";
```

#### 4.4.3 拼接视图

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

### 4.5 创建函数

#### 4.5.1 自定义函数

```sql
CREATE OR REPLACE FUNCTION ADDN(_a int,_b int) RETURNS integer AS $total$  
declare  
    total integer;  
BEGIN  
   total :=_a+_b;
   RETURN total;  
END;  
$total$ LANGUAGE plpgsql;
```


#### 4.5.2 执行动态SQL

查询执行时间大于90s的SQL，执行中断操作并保存到日志表中。无返回值

```sql
create table monitor_logs (
  query varchar(4000),
  cdate timestamp
)
```

```sql
CREATE OR REPLACE FUNCTION check_monitor() RETURNS void AS $$ DECLARE
    V_SQL VARCHAR ( 4000 );
    ITEM  RECORD;
BEGIN
    FOR ITEM IN (
        SELECT
            pid,
            query 
        FROM
            pg_stat_activity pgsa 
        WHERE
            pid != pg_backend_pid ( ) 
            AND pgsa.STATE != 'idle' 
            AND pgsa.STATE != 'idle in transaction' 
            AND pgsa.STATE != 'idle in transaction (aborted)' 
            AND EXTRACT ( epoch FROM ( now( ) - pgsa.query_start ) ) > 90 
        )
        LOOP
        V_SQL := 'SELECT pg_terminate_backend(''' || ITEM.pid || ''');';
    INSERT INTO monitor_logs ( query, cdate )
    VALUES
        ( ITEM.query, CURRENT_TIMESTAMP );
    EXECUTE V_SQL;
    END LOOP;
END;
$$ LANGUAGE PLPGSQL;
```

该函数可以配合linux定时任务实现信息系统收集

```bash
*/3 * * * * sh /home/postgres/shell/check_monitor.sh
```

```bash
#!/bin/bash
export LD_LIBRARY_PATH=/usr/local/pgsql/lib:$LD_LIBRARY_PATH
starttime=`date +'%Y-%m-%d %H:%M:%S'`
/usr/local/pgsql/bin/psql -p 5432 -d postgres -U postgres  -c "select check_monitor();"
echo '执行时间：'$starttime >> /home/postgres/shell/log.log
```

#### 4.5.3 返回游标

```sql
-- ----------------------------
-- Table structure for teacher
-- ----------------------------
DROP TABLE IF EXISTS "public"."teacher";
CREATE TABLE "public"."teacher" (
  "id" int2 NOT NULL,
  "teacher_name" varchar(50) COLLATE "pg_catalog"."default",
  "teacher_age" int2,
  "tea_salary" numeric(10,2)
);

COMMENT ON COLUMN "public"."teacher"."id" IS '主键ID';
COMMENT ON COLUMN "public"."teacher"."teacher_name" IS '教师名称';
COMMENT ON COLUMN "public"."teacher"."teacher_age" IS '教师年龄';
COMMENT ON COLUMN "public"."teacher"."tea_salary" IS '教师工资';

INSERT INTO "public"."teacher" VALUES (1, '张飞', 35, 12000.00);
INSERT INTO "public"."teacher" VALUES (2, '关羽', 12, 30000.00);
INSERT INTO "public"."teacher" VALUES (3, '杰克', 18, 40000.00);
INSERT INTO "public"."teacher" VALUES (4, '李逵', 20, 18000.00);
INSERT INTO "public"."teacher" VALUES (5, '兰陵王', 13, 7800.00);
INSERT INTO "public"."teacher" VALUES (6, '安其拉', 35, 90000.00);
INSERT INTO "public"."teacher" VALUES (7, '李白', 7, 59000.00);
INSERT INTO "public"."teacher" VALUES (8, '赵四', 9, 42000.00);
INSERT INTO "public"."teacher" VALUES (9, '王五', 5, 2000.00);
INSERT INTO "public"."teacher" VALUES (10, 'xzh', 13, 1000000.00);

ALTER TABLE "public"."teacher" ADD CONSTRAINT "teacher_pkey" PRIMARY KEY ("id");
```

```sql
CREATE OR REPLACE FUNCTION loadata ( IN pagenum int4, OUT v_total int8, OUT v_list refcursor ) 
    RETURNS record AS $$ DECLARE
    v_sql TEXT;
BEGIN
    OPEN v_list FOR 
    SELECT * FROM teacher ORDER BY ID LIMIT 5 OFFSET ( pageNum - 1 ) * 5;
        
    SELECT COUNT
        ( * ) INTO v_total 
    FROM
        teacher;
END;
$$ LANGUAGE plpgsql
```


### 4.6 创建触发器


1. 创建表

```sql
-- 学生分数表
create table stu_score
(
  stuno serial not null,       --学生编号
  major character varying(16), --专业课程
  score integer                --分数
);

-- 汇总表
create table major_stats
(
  major character varying(16), --专业课程
  total_score integer,         --总分
  total_students integer       --学生总数
);
```

2. 创建计算规则函数
   
```sql
CREATE OR REPLACE FUNCTION fun_stu_major() RETURNS TRIGGER AS $$ DECLARE
    rec record;
BEGIN
    DELETE FROM major_stats; --将统计表里面的旧数据清空
    FOR rec IN ( SELECT major, SUM(score) AS total_score, COUNT(*) AS total_students FROM stu_score GROUP BY major ) loop    
        INSERT INTO major_stats
        VALUES
            ( rec.major, rec.total_score, rec.total_students );
    END loop;
RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

3. 创建触发器

```sql
create trigger tri_stu_major 
AFTER insert or update or delete
on stu_score 
for each row
execute procedure fun_stu_major()
```



## 5. 统计信息

统计信息主要分为两类：

- `负载指标`统计信息（Monitoring stats），通过stat collector进程进行实时采集更新的负载指标，记录一些对磁盘块、表、索引相关的统计信息，SQL语句执行代价信息
- `数据分布状态描述`统计信息（Data distribution stats），这些统计信息为优化器选择最优执行计划提供依据，分为后台进程autovacuum lancher触发和手动执行analyze table进行手动采集

### 5.1 负载指标统计

#### 5.1.1 pg_stat_database

通过pg_stat_database我们可以大致的了解一个数据库的历史运行情况，比较常见的一个问题定位有

- 当tup_returned值远大于tup_fetched时，说明该数据库下存在较多全表扫描SQL，结合pg_stat_statments来定位具体慢SQL或者结合pg_stat_user_tables来定位全表扫描相关表
- 当tup_updated的数值比较大时，说明数据库有很频繁的更新，这个时候就需要关注一下vacuum相关的指标和长事务，如果没有及时进行垃圾回收会造成数据膨胀的比较厉害，一定程度会响应表查询效率
- 当temp_files的数值比较大时，说明存在很多的排序，hash，或者聚合这种操作，可以通过增大work_mem减少临时文件的产生，并且同时这些操作的性能也会有较大的提升

```sql
select * from pg_stat_database where datname='hckq';
```

```lua
datid                 | 45361                           //数据库oid
datname               | hckq                            //数据库名称
numbackends           | 58                              //访问当前数据库连接数量
xact_commit           | 56996933                        //该数据库事务提交总量
xact_rollback         | 17754                           //该数据库事务回滚总量
blks_read             | 1066706826461                   //总磁盘物理读的块数，这里read也可能是从page cache读取，如果这里很高需要结合blk_read_time看是否真的存在很多实际从磁盘读取的情况。
blks_hit              | 626755052417                    //在shared_buffer命中的块数
tup_returned          | 7744672353046                   //对于表来说是全表扫描的行数，对于索引是通过索引方法返回的索引行数，如果这个值数量明显大于tup_fetched，说明当前数据库存在大量全表扫描的情况。
tup_fetched           | 1048558532626                   //指通过索引返回的行数
tup_inserted          | 3643226                         //插入的行数
tup_updated           | 1173502495                      //更新的行数
tup_deleted           | 110792                          //删除的行数
conflicts             | 0                               //与恢复冲突取消的查询次数(只会在备库上发生)
temp_files            | 231767532                       //产生临时文件的数量，如果这个值很高说明work_mem需要调大
temp_bytes            | 65209734849881                  //临时文件的大小
deadlocks             | 430                             //死锁的数量，如果这个值很大说明业务逻辑有问题
blk_read_time         | 0                               //数据库中花费在读取文件的时间，这个值较高说明内存较小，需要频繁的从磁盘中读入数据文件
blk_write_time        | 0                               //数据库中花费在写数据文件的时间，pg中脏页一般都写入page cache，如果这个值较高，说明page cache较小，操作系统的page cache需要更积极的写入。
stats_reset           | 2023-09-06 14:12:22.841839+08   //统计信息重置的时间
```

#### 5.1.2 pg_stat_user_tables

通过pg_stat_user_tables，我们可以知道当前数据库下哪些表发生全表扫描频繁，哪些表变更比较频繁，对于变更较频繁的表可多关注其vacuum相关的指标，避免表膨胀

```sql
select * from pg_stat_user_tables where relname='t1';
```

```lua
relid               | 47798                             //表的oid
schemaname          | public                            //schema模式
relname             | t1                                //表名称
seq_scan            | 1705895                           //发生全表扫描次数
seq_tup_read        | 487983248809                      //全表扫描数据行数，如果这个值很大说明对这个表进行sql很有可能都是全表扫描，需要结合具体的执行计划来看
idx_scan            | 3073852035                        //索引扫描测试
idx_tup_fetch       | 17609938963                       //通过索引扫描返回的行数
n_tup_ins           | 17606                             //插入数据行数
n_tup_upd           | 46413                             //更新数据行数
n_tup_del           | 0                                 //删除数据行数
n_tup_hot_upd       | 20624                             //hot update的数据行数，这个值与n_tup_upd越接近说明update的性能较好，更新数据时不会更新索引。
n_live_tup          | 299112                            //活着的行数量
n_dead_tup          | 34                                //死亡的行数量,无效数据行
n_mod_since_analyze | 16746                             //上次analyze的时间
last_vacuum         |                                   //上次手动vacuum的时间
last_autovacuum     | 2023-10-09 15:10:47.935799+08     //上次autovacuum的时间
last_analyze        |                                   //上次手动analyze的时间
last_autoanalyze    | 2023-11-06 14:50:04.634755+08     //上次自动analyze的时间
vacuum_count        | 0                                 //vacuum的次数
autovacuum_count    | 1                                 //autovacuum的次数
analyze_count       | 0                                 //analyze的次数
autoanalyze_count   | 1                                 //自动analyze的次数
```

#### 5.1.3 pg_stat_user_indexes

通过pg_stat_user_indexes我们可以查看对应索引的使用情况，可以协助我们判断哪些索引当前基本不使用，对这些无效的冗余索引，可进行索引删除

```sql
select * from pg_stat_user_indexes where relname='t1';
```

```lua
relid         | 17087           //相关表的oid
indexrelid    | 17094           //索引oid
schemaname    | public          //schema模式
relname       | t1              //表名
indexrelname  | t1_pkey         //索引名
idx_scan      | 149             //通过索引扫描的次数，如果这个值很小，说明这个索引很少被用到，可以考虑进行删除
idx_tup_read  | 154             //通过任意索引方法返回的索引行数
idx_tup_fetch | 140             //通过索引方法返回的数据行数
```


```sql
-- 查询单个索引
select pg_size_pretty(pg_relation_size('index')) as size;
-- 查询所有索引
select indexrelname, pg_size_pretty(pg_relation_size(relid)) as size from pg_stat_user_indexes where schemaname='public' order by pg_relation_size(relid) desc;
-- 查询索引DDL
select * from pg_indexes where tablename='t1'; 
```

#### 5.1.4 pg_statio_user_tables

通过对pg_statio_user_tables的查询，如果heap_blks_read，idx_blks_read很高说明shared_buffer较小，存在频繁需要从磁盘或者page cache读取到shared_buffer中

```sql
select * from pg_statio_user_tables where relname='t1';
```

```lua
relid           | 17087         //相关表oid
schemaname      | public        //schema模式
relname         | t1            //表名
heap_blks_read  | 54            //指从page cache或者磁盘中读入表的块数
heap_blks_hit   | 13620         //指在shared_buffer中命中表的块数
idx_blks_read   | 16            //指从page cache或者磁盘中读入索引的块数
idx_blks_hit    | 19536         //在shared_buffer中命中的索引的块数
toast_blks_read | 0             //从page cache或者磁盘中读入toast表的块数
toast_blks_hit  | 0             //指在shared_buffer中命中toast表的块数
tidx_blks_read  | 0             //从page cache或者磁盘中读入toast表索引的块数
tidx_blks_hit   | 0             //指在shared_buffer中命中toast表索引的块数
```

#### 5.1.5 pg_stat_bgwriter

```sql
select * from pg_stat_bgwriter;
```

```lua
checkpoints_timed     | 9208            //指超过checkpoint_timeout的时间后触发的检查点
checkpoints_req       | 18              //指手动触发的检查点或者因为wal文件数量到达max_wal_size大小时也会增加，如果这个值大于checkpoints_timed，说明checkpoint_timeout设置的不合理。
checkpoint_write_time | 1404403         //指从shared_buffer中write到page cache花费的时间
checkpoint_sync_time  | 980             //指checkpoint调用fsync将脏数据同步到磁盘花费的时间，如果这个时间很长容易造成IO的抖动，这时候需要增加checkpoint_timeout或者增加checkpoint_completion_target。
buffers_checkpoint    | 19861           //checkpoint写入的脏块的数量
buffers_clean         | 1868            //通过bgwriter写入的块的数量
maxwritten_clean      | 0               //指bgwriter超过bgwriter_lru_maxpages时停止的次数，如果这个值很高说明需要增加bgwriter_lru_maxpages的大小
buffers_backend       | 404509          //通过backend写入的块数量
buffers_backend_fsync | 0               //指backend需要fsync的次数
buffers_alloc         | 54296           //被分配的缓冲区数量
stats_reset           | 2020-09-23 15:14:57.052247+08
```

#### 5.1.6 pg_stat_replication

pg_stat_replication仅仅在主从架构下才会显示相关数据。根据对pg_stat_replication表的查询可以查看当前复制的模式、复制配置信息、复制位点信息等

```sql
select * from pg_stat_replication;
```

```lua
pid              | 15040                    //负责流复制进程的pid
usesysid         | 16384                    //用户ID
usename          | repl                     //复制用户
application_name | walreceiver              //这是同步复制的通常设置，它可以通过连接字符串传递到master。
client_addr      | 192.168.1.171            //客户端地址
client_hostname  |                          //客户端主机名称，可在postgres.conf中对log_hostname参数设置，启动DNS反向查找
client_port      | 58690                    //客户端用来和WALsender进行通信使用的TPC端口号，如果不本地UNIX套接字被使用了将显示-1。
backend_start    | 2020-09-05 13:42:30.108815+08    //流复制开始时间
backend_xmin     |
state            | streaming                //复制模式
sent_lsn         | 0/3000148                //发送到连接的最后的位点
write_lsn        | 0/3000148                //standby数据库落盘的位点
flush_lsn        | 0/3000148                //standby数据库flush的位点
replay_lsn       | 0/3000148                //standby数据库重放的位点
write_lag        |                          //写延迟间隙
flush_lag        |                          //flush延迟间隙
replay_lag       |                          //重放延迟间隙
sync_priority    | 0                        //复制优先级权重
sync_state       | async                    //同步模式，同步or异步
reply_time       | 2020-09-05 13:49:41.269624+08    //
```

#### 5.2.7 pg_stat_statements

pg_stat_statements模块提供一种跟踪执行统计服务器执行的所有SQL语句的手段。该模块默认是不开启的，如果需要开启需要我们手动对其进进行编译安装，修改配置文件并重启数据库，并在使用前手动载入该模块

```sql
select * from  pg_stat_statements limit 1;
```

```lua
userid              | 10                        //用户id
dbid                | 13547                     //数据库oid
queryid             | 1194713979                //查询id
query               | SELECT * FROM pg_available_extensions WHERE name = 'pg_stat_statements'   //查询SQL
calls               | 1                         //调用次数
total_time          | 53.363875                 //SQL总共执行时间
min_time            | 53.363875                 //SQL最小执行时间
max_time            | 53.363875                 //SQL最大执行时间
mean_time           | 53.363875                 //SQL平均执行时间
stddev_time         | 0                         //SQL花费时间的表中偏差
rows                | 1                         //SQL返回或者影响的行数
shared_blks_hit     | 1                         //SQL在在shared_buffer中命中的块数
shared_blks_read    | 0                         //SQL从page cache或者磁盘中读取的块数
shared_blks_dirtied | 0                         //SQL语句弄脏的shared_buffer的块数
shared_blks_written | 0                         //SQL语句写入的块数
local_blks_hit      | 0                         //临时表中命中的块数
local_blks_read     | 0                         //临时表需要读的块数
local_blks_dirtied  | 0                         //临时表弄脏的块数
local_blks_written  | 0                         //临时表写入的块数
temp_blks_read      | 0                         //从临时文件读取的块数
temp_blks_written   | 0                         //从临时文件写入的数据块数
blk_read_time       | 0                         //从磁盘或者读取花费的时间
blk_write_time      | 0                         //从磁盘写入花费的时
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
-- 响应时间抖动最严重 SQL
select userid::regrole, dbid, query from pg_stat_statements order by stddev_time desc limit 5;  
-- 最耗共享内存 SQL
select userid::regrole, dbid, query from pg_stat_statements order by (shared_blks_hit+shared_blks_dirtied) desc limit 5;
-- 最耗临时空间 SQL
select userid::regrole, dbid, query from pg_stat_statements order by temp_blks_written desc limit 5;  
-- 清理历史统计信息
select pg_stat_statements_reset(); 

-- 执行效率
SELECT
    userid AS 执行者ID ,
    dbid AS 执行数据库ID ,
    query AS 执行的语句 ,
    calls AS 执行次数 ,
    total_time AS 执行总时间,
    total_time / calls AS 执行平均时间 ,
    ROWS AS 影响的总行数 ,
    ROWS / calls AS 影响单次行数,
    100.0 * shared_blks_hit / NULLIF ( shared_blks_hit + shared_blks_read, 0 ) AS 缓存命中率 
FROM
    pg_stat_statements 
ORDER BY
    total_time / calls DESC 
    LIMIT 100
```

压测

```bash
createdb bench              # 建压测库
pgbench -i -s 50 bench      # 初始化数据：-s 数量因子倍数，默认10万条
select pg_database_size('bench')/1024/1024||'M';        # 查看压测库大小
```

```bash
nohup pgbench -c 100 -T 20 -r bench > file.out  2>&1    # 100个session执行20s
more file.out
```

### 5.2 数据分布统计

#### 5.2.1 pg_stats

通过对pg_stats的查询，可以查看每个字段的数据分析统计信息，类似SQL Server的直方图，为优化器选择最佳执行计划提供依据，pg_stats只有管理员账号才可以访问

```sql
select * from pg_stats where tablename='t1' limit 1
```

```lua
schemaname             | public         //schema模式
tablename              | t1             //表名
attname                | id             //列名
inherited              | f              //如果为真，那么说明还字段存在包是继承而来的子字段，不只是指定表的值。
null_frac              | 0              //该字段为空记录数的百分比
avg_width              | 4              //该字段每行的平均长度
n_distinct             | -1             //如果大于零，就是在字段中唯一数值的估计数目。如果小于零， 就是唯一数值的数目被行数除的负数。用负数形式是因为ANALYZE 认为独立数值的数目是随着表增长而增长； 正数的形式用于在字段看上去好像有固定的可能值数目的情况下。比如， -1 表示一个唯一字段，独立数值的个数和行数相同。
most_common_vals       |                //该字段最常用数值的列表。如果看上去没有啥数值比其它更常见，则为 null。主键一般为null。
most_common_freqs      |                //该字段最常用数值的频率，也就是说，每个出现的次数除以行数。 如果most_common_vals是 null ，则为 null。
histogram_bounds       | {1,100,200,300,400,500,600,700,}   //一个数值的列表，它把字段的数值分成几组大致相同热门的组。
correlation            | 1              //统计与字段值的物理行序和逻辑行序有关。它的范围从 -1 到 +1 。 在数值接近 -1 或者 +1 的时候，在字段上的索引扫描将被认为比它接近零的时候开销更少， 因为减少了对磁盘的随机访问。如果字段数据类型没有<操作符，那么这个字段为null
most_common_elems      |                //经常在字段值中出现的非空元素值的列表。（标量类型为空。）
most_common_elem_freqs |                //最常见元素值的频率列表，也就是，至少包含一个给定值的实例的行的分数。 每个元素频率跟着两到三个附加的值；它们是在每个元素频率之前的最小和最大值， 还有可选择的null元素的频率。（当most_common_elems 为null时，为null）
elem_count_histogram   |                //该字段中值的不同非空元素值的统计直方图，跟着不同非空元素的平均值。（标量类型为空。）
```

#### 5.2.2 空间

1. 表占用空间

```sql
select relname, pg_size_pretty(pg_relation_size(relid)) as size from pg_stat_user_tables order by pg_relation_size(relid) DESC;
-- 表占用空间，包括索引
select relname, pg_size_pretty(pg_total_relation_size(relid)) as size from pg_stat_user_tables order by pg_relation_size(relid) desc
```

#### 5.2.3 数据

1. 所有表记录行数查询

```sql
SELECT schemaname,relname,n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;
-- 更新某个表
vacuum tablename
-- 更新该数据库所有表
vacuum               
```