# DM 8

## 1. 安装

### 1.1 二进制安装

### 1.2 docker安装

## 2. 库操作

### 2.1 表空间

```bash
create tablespace tbs1 datafile '/opt/dmdbms/data/prod/tbs1_01.dbf' size 128 autoextend on next 4 maxsize 2048; # 初始大小128m，每次自动扩充4m，最大尺寸2g
drop tablespace tbs1; 
alter tablespace tbs1 offline|online; # 表空间脱机
select tablespace_name from dba_tablespaces;
select tablespace_name,file_name from dba_data_files;
```    

### 2.2 用户授权

```bash
create user xzh identified by 123456789 limit password_life_time 60, failed_login_attemps 5, password_lock_time 5;
drop user xzh cascade;
create user xzh identified by "123456789" default tablespace tbs1 temporary tablespace temp_tbs1;
grant resource,public,soi,svi,vti to xzh;
```

### 2.3 备份恢复

### 2.4 数据迁移

## 3. 表操作

## 4. PLSQL

