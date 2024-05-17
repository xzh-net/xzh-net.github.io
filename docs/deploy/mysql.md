# MySQL 5.7.44

## 1. 安装

### 1.1 单机

#### 1.1.1 卸载mariadb

```bash
rpm -qa|grep mariadb
rpm -qa | grep -i mysql
rpm -e mysql-community-server-5.7.44-1.el7.x86_64 --nodeps
```

#### 1.1.2 上传解压

```bash
cd /opt/software/
mkdir -p mysql
tar xvf mysql-5.7.44-1.el7.x86_64.rpm-bundle.tar -C /opt/software/mysql
```

#### 1.1.3 执行安装

```bash
yum -y install libaio
cd /opt/software/mysql
rpm -ivh mysql-community-common-5.7.44-1.el7.x86_64.rpm mysql-community-libs-5.7.44-1.el7.x86_64.rpm mysql-community-client-5.7.44-1.el7.x86_64.rpm mysql-community-server-5.7.44-1.el7.x86_64.rpm
```

#### 1.1.4 修改配置

```bash
vim /etc/my.cnf

datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

#### 1.1.5 初始化

```bash
mysqld --initialize                         # 初始化mysql
chown mysql:mysql /var/lib/mysql -R         # 更改所属组
cat /var/log/mysqld.log | grep password     # 初始密码
systemctl start mysqld                      # 启动mysql
systemctl enable mysqld                     # 设置开机启动
ps aux | grep mysqld 
```

#### 1.1.6 客户端测试

```bash
mysql -u root -p
set password = password('123456');
grant all privileges on *.* to 'root' @'%' identified by '123456';
flush privileges;
```

#### 1.1.7 完全卸载

1. 停止服务

```bash
systemctl stop mysqld
```

2. 删除依赖

```bash
rpm -qa | grep -i mysql 
yum remove -y mysql-community-libs-5.7.44-1.el7.x86_64 mysql-community-common-5.7.44-1.el7.x86_64 mysql-community-client-5.7.44-1.el7.x86_64 mysql-community-server-5.7.44-1.el7.x86_64
rpm -qa | grep -i mysql     # 确认删除依赖
```

3. 删除配置

```bash
find / -name mysql

# 删除目录
rm -rf /var/lib/mysql
rm -rf /var/lib/mysql/mysql
rm -rf /usr/share/mysql
# 删除默认配置日志
rm -rf /etc/my.cnf
rm -rf /var/log/mysqld.log
```

## 2. 库操作

### 2.1 用户管理

1. 创建用户

```sql
mysql -uroot -p123456
CREATE DATABASE IF NOT EXISTS sonar DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
GRANT ALL ON sonar.* TO 'sonar'@'%' IDENTIFIED BY '123456';
GRANT ALL ON sonar.* TO 'sonar'@'localhost' IDENTIFIED BY '123456';
GRANT PROCESS ON *.* TO 'sonar'@'%';
flush privileges;
```

> 使用root给其他用户授权必须进到服务器执行

2. 设置密码

```sql
--方法1，密码实时更新；修改用户“test”的密码为“1122”
set password for test =password('1122');
--方法2，需要刷新；修改用户“test”的密码为“1234”
update  mysql.user set  password=password('1234')  where user='test'
--刷新
flush privileges;
```

3. 删除用户

```sql
drop user sonar;
drop database sonar;
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
vim /etc/my.cnf

lower_case_table_names=1
service mysql restart
```

### 2.3 查询日志

```bash
set global slow_query_log = on      # 临时开启慢查询日志
set global slow_query_log = off     # 临时关闭
set long_query_time = 1             # 临时设置查询临界点
set globle log_output = file        # 设置慢查询存储的方式
show variables like '%quer%'        # 开启状态和慢查询日志储存的位置

cat -n  /data/mysql/mysql-slow.log  # 查看示例
```

### 2.4 审计日志

### 2.5 归档日志

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

### 2.6 备份恢复

```bash
mysqldump -uroot -proot --all-databases >/tmp/all.sql                               # 备份所库
mysqldump -uroot -proot --databases db1 db2 >/tmp/user.sql                          # 备份指定库
mysqldump -uroot -proot --databases db1 --tables a1 --where='id=1'  >/tmp/a1.sql    # 备份指定库指定表
mysqldump -uroot -proot --no-data --databases db1 >/tmp/db1.sql                     # 只导指定库的表结构
mysqldump --set-gtid-purged=OFF -h 127.0.0.1 -u root -p 123456 dbname --ignore-table=dbname.tb1 --ignore-table=dbname.tb2 > /tmp/all.sql   # 忽略表
mysql -uroot -proot -h 127.0.0.1 -P 3306 sonar</tmp/all.sql                         # 导入
```

## 3. 表操作

### 3.1 表结构
   
```sql
-- 获取所有表信息
SELECT
    table_name,
    ENGINE,
    table_comment,
    create_time 
FROM
    information_schema.TABLES 
WHERE
    table_schema = (
    SELECT DATABASE
    ()) 
    AND table_name = 'tableName' 
ORDER BY
    create_time DESC

-- 获取指定表结构信息
SELECT
    TABLE_SCHEMA AS '库名',
    TABLE_NAME AS '表名',
    COLUMN_NAME AS '列名',
    ORDINAL_POSITION AS '列的排列顺序',
    COLUMN_DEFAULT AS '默认值',
    IS_NULLABLE AS '是否为空',
    DATA_TYPE AS '数据类型',
    CHARACTER_MAXIMUM_LENGTH AS '字符最大长度',
    NUMERIC_PRECISION AS '数值精度(最大位数)',
    NUMERIC_SCALE AS '小数精度',
    COLUMN_TYPE AS '列类型',
    COLUMN_KEY 'KEY',
    EXTRA AS '额外说明',
    COLUMN_COMMENT AS '注释' 
FROM
    information_schema.COLUMNS 
WHERE
    TABLE_NAME = 'tableName' 
ORDER BY
    TABLE_NAME,
    ORDINAL_POSITION
```

## 4. 统计信息

### 4.1 负载指标统计

### 4.2 数据分布统计

#### 4.2.1 空间

1. 表占用空间

```sql
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS total_size_mb
FROM
    information_schema.tables
WHERE
    table_schema = 'database_name'
order by ROUND(((data_length + index_length) / 1024 / 1024), 2)
```

2. 所有表占用空间

```sql
select table_schema as 'database', SUM(ROUND((data_length + index_length) / 1024 / 1024, 2)) AS 'Size(MB)' FROM information_schema.tables where table_schema = 'database_name'
```

#### 4.2.2 数据

1. 每个表记录行数查询

```sql
select table_name,table_rows from information_schema.tables where TABLE_SCHEMA = 'database_name' order by table_rows desc;
```

## 5. SQL

### 5.1 FUNCTION

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

### 5.2 PROCEDURE

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