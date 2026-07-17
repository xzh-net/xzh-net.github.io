# MySQL 5.7.44

- 下载地址：https://dev.mysql.com/downloads/mysql/

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

### 2.1 创建用户

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

### 2.3 慢查询日志

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

### 2.6 数据备份

```bash
mysqldump -uroot -proot --all-databases >/tmp/all.sql                               # 备份所库
mysqldump -uroot -proot --databases db1 db2 >/tmp/user.sql                          # 备份指定库
mysqldump -uroot -proot --databases db1 --tables a1 --where='id=1'  >/tmp/a1.sql    # 备份指定库指定表
mysqldump -uroot -proot --no-data --databases db1 >/tmp/db1.sql                     # 只导指定库的表结构
mysqldump --set-gtid-purged=OFF -h 127.0.0.1 -u root -p 123456 dbname --ignore-table=dbname.tb1 --ignore-table=dbname.tb2 > /tmp/all.sql   # 忽略表
# 还原
mysql -uroot -proot -h 127.0.0.1 -P 3306 sonar</tmp/all.sql                         # 导入
```

## 3. 表操作

### 3.1 查询表结构
   
获取所有表信息

```sql
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
```

获取指定表结构信息

```sql
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

### 3.2 创建函数

1. 使用聚合函数拼接

```sql
CREATE FUNCTION `F_ACTUSER`(v_FLOWCID VARCHAR(40)) 
RETURNS LONGTEXT CHARSET utf8
BEGIN
    DECLARE v_STR LONGTEXT DEFAULT '';
    
    SELECT GROUP_CONCAT(DISTINCT B.USERNAME ORDER BY B.USERNAME SEPARATOR ',')
    INTO v_STR
    FROM TS_FLOW_PATH_COM A
    INNER JOIN VJSP_USERS B ON B.USERID = A.TS_MK_USERID
    WHERE A.FLOWCID = v_FLOWCID
      AND A.FLOWZT NOT IN (-1, 2, 3);
      
    RETURN v_STR;
END;
```

2. 基于 Twitter Snowflake 算法实现的分布式唯一 ID 生成器

```sql
-- 1. 初始化用户变量，用于在函数调用间保持状态
SET @last_timestamp = -1;
SET @sequence = 0;

-- 2. 删除已存在的同名函数（如果有）
DROP FUNCTION IF EXISTS generate_snowflake_id;

-- 3. 创建新的函数
DELIMITER //

CREATE FUNCTION generate_snowflake_id()
RETURNS BIGINT
READS SQL DATA
BEGIN
    DECLARE timestamp BIGINT;
    -- 机器ID，你可以根据实际情况调整 (范围: 0-31)
    DECLARE machine_id BIGINT DEFAULT 1;
    -- 数据中心ID，你可以根据实际情况调整 (范围: 0-31)
    DECLARE data_center_id BIGINT DEFAULT 0;
    -- 起始时间戳 (2010-01-01 00:00:00 UTC)，可根据需要调整
    DECLARE epoch BIGINT DEFAULT 1288834974657;

    -- 获取当前毫秒级时间戳，并减去起始时间
    SET timestamp = FLOOR(UNIX_TIMESTAMP(NOW(3)) * 1000) - epoch;

    -- 序列号生成逻辑
    IF timestamp = @last_timestamp THEN
        -- 同一毫秒内，序列号加1，并确保在0-4095之间循环
        SET @sequence = (@sequence + 1) % 4096;
    ELSE
        -- 新的毫秒，重置序列号
        SET @sequence = 0;
    END IF;

    -- 更新"最后时间戳"变量
    SET @last_timestamp = timestamp;

    -- 通过位运算组合最终ID
    -- timestamp左移22位，data_center_id左移17位，machine_id左移12位，最后与序列号进行按位或操作
    RETURN (timestamp << 22) | (data_center_id << 17) | (machine_id << 12) | @sequence;
END //

DELIMITER ;
```

测试
```sql
select generate_snowflake_id()
```

### 3.3 创建存储过程

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
