# DM 8

## 1. 安装

### 1.1 准备工作

环境：CentOS7_x86_64，数据库版本：dm8_rh7_64_ent_8.1.1.87，安装前必须创建dmdba用户，禁止使用root用户安装数据库。

#### 1.1.1 创建用户

```bash
groupadd dinstall
useradd -g dinstall -m -d /home/dmdba -s /bin/bash dmdba
passwd dmdba
```

#### 1.1.2 修改文件打开最大数

永久生效

```bash
vi /etc/security/limits.conf

dmdba hard nofile 65536
dmdba soft nofile 65536
dmdba hard stack 32768
dmdba soft stack 16384
```

切换到 dmdba 用户，查看是否生效，命令如下：

```bash
su - dmdba
ulimit -a
```

临时生效

```bash
ulimit -n 65536
```

#### 1.1.3 挂载镜像

切换到 root 用户，将 DM 数据库的 iso 安装包保存在任意位置，例如 /opt 目录下，执行如下命令挂载镜像：

```bash
cd /opt
unzip dm8_20230418_x86_rh6_64.zip
mkdir /mnt
mount -o loop /opt/dm8_20230418_x86_rh6_64.iso /mnt
```

#### 1.1.4 创建安装目录

使用 root 用户建立文件夹，待 dmdba 用户建立完成后需将文件所有者更改为 dmdba 用户，否则无法安装到该目录下

```bash
mkdir /dm8
```

#### 1.1.5 修改安装目录权限

将新建的安装路径目录权限的用户修改为 dmdba，用户组修改为 dinstall。命令如下：

```bash
chown dmdba:dinstall -R /dm8/
chmod -R 755 /dm8   # 给安装路径下的文件设置 755 权限
```

### 1.2 数据库安装

DM 数据库在 Linux 环境下支持命令行安装和图形化安装

#### 1.2.1 命令行安装

```bash
su - dmdba
cd /mnt/
./DMInstall.bin -i
```

按需求选择安装语言，默认为中文。本地安装选择【不输入 Key文件】，选择【默认时区 21】。

![](../../assets/_images/deploy/dm/choose-lang-time.png)

选择【1-典型安装】，按已规划的安装目录 /dm8 完成数据库软件安装，不建议使用默认安装目录。

![](../../assets/_images/deploy/dm/choose-type-path.png)

数据库安装大概 1~2 分钟，数据库安装完成后，显示如下界面。

![](../../assets/_images/deploy/dm/install-success.png)

数据库安装完成后，需要切换至 root 用户执行上图中的命令 /dm8/script/root/root_installer.sh 创建 DmAPService，否则会影响数据库备份

#### 1.2.2 配置环境变量

切换到 root 用户进入 dmdba 用户的根目录下，配置对应的环境变量。DM_HOME 变量和动态链接库文件的加载路径在程序安装成功后会自动导入。命令如下：

```bash
cd /home/dmdba/
vim .bash_profile

export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/dm8/bin"
export DM_HOME="/dm8"
export PATH=$PATH:$DM_HOME/bin:$DM_HOME/tool
```

![](../../assets/_images/deploy/dm/dm-home-path.png)

切换至 dmdba 用户下，执行以下命令，使环境变量生效。

```bash
su - dmdba
source .bash_profile
```

### 1.3 配置实例

使用 dmdba 用户配置实例，进入到 DM 数据库安装目录下的 bin 目录中，使用 dminit 命令初始化实例

```bash
cd /dm8/bin
./dminit help
```

![](../../assets/_images/deploy/dm/ml-licence-dminithelp.png)

需要注意的是页大小 (page_size)、簇大小 (extent_size)、大小写敏感 (case_sensitive)、字符集 (charset) 这四个参数，一旦确定无法修改，需谨慎设置
   - extent_size 指数据文件使用的簇大小，即每次分配新的段空间时连续的页数。只能是 16 页或 32 页或 64 页之一，缺省使用 16 页。
   - page_size 数据文件使用的页大小，可以为 4 KB、8 KB、16 KB 或 32 KB 之一，选择的页大小越大，则 DM 支持的元组长度也越大，但同时空间利用率可能下降，缺省使用 8 KB。
   - case_sensitive 标识符大小写敏感，默认值为 Y 。当大小写敏感时，小写的标识符应用双引号括起，否则被转换为大写；当大小写不敏感时，系统不自动转换标识符的大小写，在标识符比较时也不区分大小写，只能是 Y、y、N、n、1、0 之一。
   - charset 字符集选项。0 代表 GB18030；1 代表 UTF-8；2 代表韩文字符集 EUC-KR；取值 0、1 或 2 之一。默认值为 0。

可以使用默认参数初始化实例，需要附加实例存放路径。此处以初始化实例到 /dm8/data 目录下为例，初始化命令如下：

```bash
./dminit path=/dm8/data
```

### 1.4 注册服务

注册服务需使用 root 用户进行注册。使用 root 用户进入数据库安装目录的 /dm8/script/root 下，如下所示：

```bash
cd /dm8/script/root
./dm_service_installer.sh -t dmserver -dm_ini /dm8/data/DAMENG/dm.ini -p DMSERVER
```

用户可根据自己的环境更改 dm.ini 文件的路径以及服务名，如下所示：

```bash
./dm_service_installer.sh -h
```

### 1.5 启停服务

```bash
systemctl start DmServiceDMSERVER.service
systemctl stop DmServiceDMSERVER.service
systemctl restart DmServiceDMSERVER.service
systemctl status DmServiceDMSERVER.service
```

### 1.6 客户端测试

SYSDBA/SYSDBA

## 2. 库操作

### 2.1 创建表空间

表空间名称系统默认全部转换成大写，如有特殊需求使用小写，需加双引号。临时表空间仅初始化时自动创建，无法手动新增。

```sql
-- 初始大小128m，每次自动扩充10m，最大尺寸2g
create tablespace tbs1 datafile '/dm8/data/DAMENG/tbs1.dbf' size 128 autoextend on next 10 maxsize 2048; 
-- 修改表空间，打开自动扩展，每次自动扩展 100M ，扩展上限 10240M
alter tablespace "tbs1" datafile '/dm8/data/DAMENG/tbs1.dbf' autoextend on next 100 maxsize 10240;

-- 查询表空间
SELECT TABLESPACE_NAME FROM DBA_TABLESPACES;
-- 查询表空间文件位置
SELECT TABLESPACE_NAME,FILE_NAME FROM DBA_DATA_FILES;

-- 检查表对象
SELECT TABLE_NAME FROM DBA_TABLES WHERE TABLESPACE_NAME = 'tbs1';
-- 检查索引对象
SELECT INDEX_NAME FROM DBA_INDEXES WHERE TABLESPACE_NAME = 'tbs1';
-- 删除表空间，需要手动删除物理文件
DROP TABLESPACE tbs1;

-- 将表空间离线
ALTER TABLESPACE tbs1 OFFLINE;
-- 将表空间重新上线
ALTER TABLESPACE tbs1 ONLINE;
```

### 2.2 创建用户

1. 创建用户

基础用户
```sql
CREATE USER test2026 IDENTIFIED BY "Pwd@2026"
```

指定表空间和索引表空间
```sql
CREATE USER test2026 IDENTIFIED BY "Pwd@2026" default tablespace tbs1 default index tablespace tbs2;
```

带密码策略的用户，密码有效期（天）：60 ，最大失败登录次数：5 ，密码锁定时间（天）：5
```sql
create user test2026 identified by "Pwd@2026" limit password_life_time 60, failed_login_attemps 5, password_lock_time 5;
```

2. 删除用户

```sql
DROP USER TEST2026 CASCADE;
```

3. 查询用户

```sql
SELECT * FROM DBA_USERS;
```

4. 修改密码


5. 授权

```sql
-- 授权RESOURCE角色
GRANT RESOURCE to TEST2026;
-- 查询用户角色
SELECT * FROM DBA_ROLE_PRIVS WHERE GRANTEE = 'TEST2026';
-- 查询预定义角色
SELECT * FROM DBA_SYS_PRIVS WHERE GRANTEE='PUBLIC';
```


## 3. 表操作

### 3.1 创建表

在模式 TEST2026 下创建表 CITY，并插入数据。示例语句如下所示：

```sql
CREATE TABLE TEST2026.city
(
 city_id CHAR(3) NOT NULL,
 city_name VARCHAR(40) NULL,
 region_id INT NULL
);
```

```sql
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('BJ','北京',1);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('SJZ','石家庄',1);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('SH','上海',2);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('NJ','南京',2);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('GZ','广州',3);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('HK','海口',3);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('WH','武汉',4);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('CS','长沙',4);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('SY','沈阳',5);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('XA','西安',6);
INSERT INTO TEST2026.city(city_id,city_name,region_id) VALUES('CD','成都',7);
```


### 3.2 创建视图

对 CITY 表创建一个`只读视图`，命名为 V_CITY，保存 region_id 小于 4 的数据，列名有：city_id，city_name，region_id。示例语句如下所示：

```sql
CREATE OR REPLACE VIEW TEST2026.v_city AS
SELECT
        city_id  ,
        city_name ,
        region_id
FROM
        TEST2026.city
WHERE
        region_id < 4
WITH READ ONLY;
```

删除视图

```sql
DROP VIEW TEST2026.V_CITY;
```

创建一个可`更新视图`。以后对该视图作插入、修改和删除操作时，系统均会自动用 WHERE 后的条件作检查，不满足条件的数据，则不能通过该视图更新相应基表中的数据

```sql
CREATE OR REPLACE VIEW TEST2026.v_city AS
SELECT
        city_id  ,
        city_name ,
        region_id
FROM
        TEST2026.city
WHERE
        region_id < 4
WITH CHECK OPTION;
```

### 3.3 创建存储过程

创建测试表
```sql
CREATE TABLE TEST_TAB (ID INT PRIMARY KEY, NAME VARCHAR(30));
```

创建有参数存储过程 p_test
```sql
CREATE OR REPLACE PROCEDURE P_TEST(I IN INT)
AS
    J INT;
BEGIN
    FOR J IN 1 .. I LOOP
        INSERT INTO TEST_TAB VALUES (J, 'P_TEST'||J);
    END LOOP;
END;
```

执行过程
```sql
P_TEST(3);
```

查看结果
```sql
select * from TEST_TAB;
```


### 3.4 创建函数

根据身份证号判断性别

```sql
CREATE OR REPLACE FUNCTION TEST2026.GET_SEX(id_card IN VARCHAR(50))
RETURN CHAR(2)
AS
    v_sex CHAR(2);
BEGIN
    IF to_number(substr(id_card,17,1))%2=1 THEN
     v_sex:= '男';
    ELSE  
     v_sex:= '女';
    END IF;
    RETURN v_sex;
END;
```

使用函数
```sql
-- 女
select GET_SEX(110101199612123801) from DUAL;
-- 男
select GET_SEX(710000199608051059) from DUAL;
```

### 3.5 创建序列

创建序列 SEQ_QUANTITY，起始值为 5，增量值为 2，最大值为 200。示例语句如下所示：

```sql
CREATE SEQUENCE TEST2026.seq_quantity START WITH 5 INCREMENT BY 2 MAXVALUE 200;
```

查询下一个值
```sql
SELECT TEST2026.seq_quantity.nextval FROM dual;
```

### 3.6 创建触发器

#### 3.6.1 表触发器

每次修改用户信息，记录变更审计日志

1. 创建表

```sql
-- 创建用户表
CREATE TABLE USER_INFO (
    USER_ID INT PRIMARY KEY,
    USER_NAME VARCHAR(50) NOT NULL,
    GENDER CHAR(1) CHECK (GENDER IN ('M', 'F')),
    CREATE_TIME DATETIME DEFAULT SYSDATE,
    IS_DISABLED CHAR(1) DEFAULT '0' CHECK (IS_DISABLED IN ('0', '1'))
);

COMMENT ON TABLE USER_INFO IS '用户信息表';
COMMENT ON COLUMN USER_INFO.USER_ID IS '用户ID';
COMMENT ON COLUMN USER_INFO.USER_NAME IS '用户姓名';
COMMENT ON COLUMN USER_INFO.GENDER IS '性别（M:男, F:女）';
COMMENT ON COLUMN USER_INFO.CREATE_TIME IS '创建时间';
COMMENT ON COLUMN USER_INFO.IS_DISABLED IS '是否禁用（0:否, 1:是）';

-- 创建日志记录表
CREATE TABLE USER_AUDIT_LOG (
    AUDIT_ID INT IDENTITY(1, 1) PRIMARY KEY,
    OPER_TIME DATETIME DEFAULT SYSDATE,
    OPER_TYPE VARCHAR(10) NOT NULL,
    OLD_VALUE VARCHAR(2000),
    NEW_VALUE VARCHAR(2000)
);

COMMENT ON TABLE USER_AUDIT_LOG IS '用户操作审计日志表';
COMMENT ON COLUMN USER_AUDIT_LOG.AUDIT_ID IS '审计ID（自增主键）';
COMMENT ON COLUMN USER_AUDIT_LOG.OPER_TIME IS '操作时间';
COMMENT ON COLUMN USER_AUDIT_LOG.OPER_TYPE IS '操作类型';
COMMENT ON COLUMN USER_AUDIT_LOG.OLD_VALUE IS '原值';
COMMENT ON COLUMN USER_AUDIT_LOG.NEW_VALUE IS '新值';
```

2. 创建触发器

```sql
CREATE OR REPLACE TRIGGER TRI_USER_AUDIT
AFTER INSERT OR UPDATE OR DELETE ON USER_INFO
FOR EACH ROW
BEGIN
    -- 声明变量
    DECLARE
        v_old_val VARCHAR(2000);
        v_new_val VARCHAR(2000);
        v_oper_type VARCHAR(10);
    BEGIN
        -- 判断操作类型并设置变量
        IF INSERTING THEN
            v_oper_type := '新增';
            v_old_val := NULL;
            -- 拼接新值
            v_new_val := '用户ID:' || :NEW.USER_ID || 
                        ', 姓名:' || :NEW.USER_NAME || 
                        ', 性别:' || :NEW.GENDER || 
                        ', 创建时间:' || TO_CHAR(:NEW.CREATE_TIME, 'YYYY-MM-DD HH24:MI:SS') || 
                        ', 是否禁用:' || :NEW.IS_DISABLED;
        ELSIF UPDATING THEN
            v_oper_type := '修改';
            -- 拼接原值
            v_old_val := '用户ID:' || :OLD.USER_ID || 
                        ', 姓名:' || :OLD.USER_NAME || 
                        ', 性别:' || :OLD.GENDER || 
                        ', 创建时间:' || TO_CHAR(:OLD.CREATE_TIME, 'YYYY-MM-DD HH24:MI:SS') || 
                        ', 是否禁用:' || :OLD.IS_DISABLED;
            -- 拼接新值
            v_new_val := '用户ID:' || :NEW.USER_ID || 
                        ', 姓名:' || :NEW.USER_NAME || 
                        ', 性别:' || :NEW.GENDER || 
                        ', 创建时间:' || TO_CHAR(:NEW.CREATE_TIME, 'YYYY-MM-DD HH24:MI:SS') || 
                        ', 是否禁用:' || :NEW.IS_DISABLED;
        ELSIF DELETING THEN
            v_oper_type := '删除';
            -- 拼接原值
            v_old_val := '用户ID:' || :OLD.USER_ID || 
                        ', 姓名:' || :OLD.USER_NAME || 
                        ', 性别:' || :OLD.GENDER || 
                        ', 创建时间:' || TO_CHAR(:OLD.CREATE_TIME, 'YYYY-MM-DD HH24:MI:SS') || 
                        ', 是否禁用:' || :OLD.IS_DISABLED;
            v_new_val := NULL;
        END IF;
        
        -- 插入审计日志记录
        INSERT INTO USER_AUDIT_LOG (OPER_TYPE, OLD_VALUE, NEW_VALUE)
        VALUES (v_oper_type, v_old_val, v_new_val);
    END;
END;
```

3. 测试触发器

```sql
-- 1. 测试INSERT操作
INSERT INTO USER_INFO (USER_ID, USER_NAME, GENDER, IS_DISABLED) 
VALUES (1, '张三', 'M', '0');

-- 2. 测试UPDATE操作
UPDATE USER_INFO 
SET USER_NAME = '张三丰', IS_DISABLED = '1' 
WHERE USER_ID = 1;

-- 3. 测试DELETE操作
DELETE FROM USER_INFO WHERE USER_ID = 1;

-- 4. 查看审计日志
SELECT * FROM USER_AUDIT_LOG ORDER BY OPER_TIME DESC;

-- 5. 清空测试数据
TRUNCATE TABLE USER_INFO;
TRUNCATE TABLE USER_AUDIT_LOG;
```

#### 3.6.2 管理触发器

每个触发器创建成功后都自动处于允许状态 (ENABLE)，当不想被触发，但是又不想删除这个触发器。这时，可将其设置关闭触发器 (DISABLE)。

```sql
-- 禁用触发器
ALTER TRIGGER TRI_USER_AUDIT DISABLE;
-- 启用触发器
ALTER TRIGGER TRI_USER_AUDIT ENABLE;
-- 删除触发器
DROP TRIGGER TRI_USER_AUDIT;
```

查看触发器

```sql
--查看当前数据库的全部触发器
SELECT * FROM DBA_TRIGGERS;
--查看当前用户有权限访问的触发器
SELECT * FROM ALL_TRIGGERS;
--查看示当前用户所拥有的触发器
SELECT * FROM USER_TRIGGERS;
```

### 3.7 创建链接

在主机 A 上建表 test

```sql
CREATE TABLE TEST(C1 INT,C2 VARCHAR(20));
```

在 B 上建立到 A 的数据库链接 LINK01，使用链接进行插入

```sql
CREATE LINK LINK01 CONNECT 'DAMENG' WITH "SYSDBA" IDENTIFIED BY "SYSDBA" USING '192.168.1.110/5236';
INSERT INTO TEST@LINK01 VALUES(1,'A');
COMMIT;
```

删除链接
```sql
DROP LINK LINK01;
```

### 3.8 匿名块

#### 3.8.1 执行动态SQL

打印区域id=2的数据条数

```sql
DECLARE
    v_sql VARCHAR2(1000);
    v_region_id NUMBER := 2;
    v_city_count NUMBER;
BEGIN
    -- 动态SQL
    v_sql := 'SELECT COUNT(*) FROM city WHERE region_id = :1';
    
    EXECUTE IMMEDIATE v_sql INTO v_city_count USING v_region_id;
    
    DBMS_OUTPUT.PUT_LINE('区域id=' || v_region_id || '的数据有' || v_city_count || '条');
END;
```


#### 3.8.2 调用带参数的存储过程

创建存储过程

```sql
CREATE OR REPLACE PROCEDURE proc_calculate_salary(
    p_in_param IN NUMBER,               -- 仅输入
    p_in_out_param IN OUT NUMBER,       -- 输入输出
    p_out_param OUT VARCHAR2            -- 仅输出
)
AS
BEGIN
    -- 使用并修改IN OUT参数
    p_in_out_param := p_in_out_param * 1.2;
    
    -- 设置输出参数
    p_out_param := '处理完成，IN_OUT参数已更新';
END;
```

执行存储过程

```sql
DECLARE
    v_result  NUMBER := 1000;
    v_message VARCHAR2(100);
BEGIN
    -- 调用存储过程
    proc_calculate_salary(1000, v_result, v_message);
    DBMS_OUTPUT.PUT_LINE('结果: ' || v_result || ', 消息: ' || v_message);
END;
```