# DataX 3.0

DataX 是阿里巴巴开源的一个异构数据源离线同步工具，致力于实现包括关系型数据库(MySQL、Oracle 等)、HDFS、Hive、ODPS、HBase、FTP 等各种异构数据源之间稳定高效的数据同步功能

## 1. 下载解压

源码地址：https://github.com/alibaba/DataX

http://datax-opensource.oss-cn-hangzhou.aliyuncs.com/datax.tar.gz

```bash
cd /opt/software
tar -zxvf datax.tar.gz -C /opt/
```

## 2. 运行

```bash
cd /opt/datax
bin/datax.py job/job.json
```

## 3. 基本使用

### 3.1 从stream读取数据并打印到控制台

### 3.2 Mysql导入数据到HDFS
### 3.3 Mysql导入数据到Hbase
### 3.4 Mysql数据导出到Mysql


### 3.5 Oracle导入数据到HDFS
### 3.6 Oracle导入数据到Mysql


### 3.7 MongoDB导入数据到HDFS
### 3.8 MongoDB导入数据到Mysql


### 3.9 SQLServer导入数据到HDFS
### 3.10 SQLServer导入到Mysql


### 3.11 DB2导入数据到HDFS
### 3.12 DB2导入数据到Mysql


### 3.13 Hbase导入数据到HDFS
### 3.14 Hbase导入数据到Mysql