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

```bash
python /opt/datax/bin/datax.py -r streamreader -w streamwriter  # 模板
cd /opt/datax/job
vi stream2stream.json
```

```xml
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "streamreader", 
                    "parameter": {
                        "column": [
                            {
                                "type":"string",
                                "value":"xuzhihao"
                            },
                            {
                                "type": "long",
                                "value": "2022"
                            },
                            {
                                "type":"string",
                                "value":"你好，我们都是好孩子"
                            }
                        ], 
                        "sliceRecordCount": "10"
                    }
                }, 
                "writer": {
                    "name": "streamwriter", 
                    "parameter": {
                        "encoding": "utf-8", 
                        "print": true
                    }
                }
            }
        ], 
        "setting": {
            "speed": {
                "channel": "1"
            }
        }
    }
}
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/stream2stream.json
```

### 3.2 Mysql导入数据到HDFS

```bash
python /opt/datax/bin/datax.py -r mysqlreader -w hdfswriter  # 模板
cd /opt/datax/job
vi mysql2hdfs.json
```

```xml
{
    "job": {
            {
                "reader": {
                    "name": "mysqlreader",
                    "parameter": {
                        "column": [
                            "id", 
                            "name"
                        ],  
                        "connection": [
                            {
                                "jdbcUrl": [
                                    "jdbc:mysql://172.17.17.137:3306/mall"
                                ],  
                                "table": [
                                    "pms_product"
                                ]   
                            }   
                        ],  
                        "password": "root",
                        "username": "root"
                    }   
                },  
                "writer": {
                    "name": "hdfswriter",
                    "parameter": {
                        "column": [
                            {
                                "name":"id",
                                "type":"int"
                            },  
                            {
                                "name":"name",
                                "type":"string"
                            }   
                        ],  
                        "defaultFS": "hdfs://bbfc6fd4f77c:8020",
                        "fieldDelimiter": "\t", 
                        "fileName": "pms_product.txt",
                        "fileType": "text", 
                        "path": "/datax", 
                        "writeMode": "append"
                    }   
                }   
            }   
        ],  
        "setting": {
            "speed": {
                "channel": "1"
            }
        }
    }
}
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/mysql2hdfs.json
```

### 3.3 HDFS导入数据到Mysql

```bash
python /opt/datax/bin/datax.py -r hdfsreader -w mysqlwriter  # 模板
cd /opt/datax/job
vi hdfs2mysql.json
```

```xml
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "hdfsreader", 
                    "parameter": {
                        "column": [
                            "*"
                        ], 
                        "defaultFS": "hdfs://bbfc6fd4f77c:8020", 
                        "fieldDelimiter": "\t", 
                        "path": "/datax/pms_product.txt",
                        "fileType": "text", 
                        "encoding": "UTF-8"
                    }
                }, 
                "writer": {
                    "name": "mysqlwriter", 
                    "parameter": {
                        "column": [
                            "id",
                            "name"
                        ],
                        "connection": [
                            {
                                "jdbcUrl": "jdbc:mysql://172.17.17.137:3306/mall", 
                                "table": ["pms_product_bak"]
                            }
                        ], 
                        "password": "root",  
                        "username": "root", 
                        "writeMode": "insert"
                    }
                }
            }
        ], 
        "setting": {
            "speed": {
                "channel": "1"
            }
        }
    }
}
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/hdfs2mysql.json
```

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