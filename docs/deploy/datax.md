# DataX 3.0

DataX 是阿里巴巴开源的一个异构数据源离线同步工具，致力于实现包括关系型数据库(MySQL、Oracle 等)、HDFS、Hive、ODPS、HBase、FTP 等各种异构数据源之间稳定高效的数据同步功能

- 源码地址：https://github.com/alibaba/DataX
- 下载地址：https://datax-opensource.oss-cn-hangzhou.aliyuncs.com/202309/datax.tar.gz

## 1. 安装

### 1.1 上传解压

```bash
cd /opt/software
tar -zxvf datax.tar.gz -C /opt/
```

启动服务

```bash
cd /opt/datax
bin/datax.py job/job.json
```

## 2. 基本使用

### 2.1 从stream读取数据并打印到控制台

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
                                "value":"xzh"
                            },
                            {
                                "type": "long",
                                "value": "2022"
                            },
                            {
                                "type":"string",
                                "value":"我们都是好孩子"
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

### 2.2 HDFS导入数据到Mysql

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
                        "defaultFS": "hdfs://hadoop2:9000", 
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

### 2.3 Mysql导入数据到HDFS

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
                        "defaultFS": "hdfs://hadoop2:9000",
                        "fieldDelimiter": "\t", 
                        "fileName": "mysql_pms_product.txt",
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

### 2.4 Mysql导入数据到Hbase

```bash
python /opt/datax/bin/datax.py -r mysqlreader -w hbase11xwriter  # 模板
cd /opt/datax/job
vi mysql2hbase.json
```

```xml
{
    "job": {
        "content": [{
            "reader": {
                "name": "mysqlreader",
                "parameter": {
                    "password": "root",
                    "username": "root",
                    "connection": [{
                        "jdbcUrl": ["jdbc:mysql://172.17.17.137:3306/mall"], 
                        "querySql": [
                            "select id, name, create_time from pms_product_bak where brand_name is not null"
                        ]
                    }]
                }
            },
            "writer": {
                "name": "hbase11xwriter",
                "parameter": {
                    "mode": "normal",
                    "table": "test",
                    "column": [{
                        "name": "c1:id",
                        "type": "string",
                        "index": 0
                    }, {
                        "name": "c1:name",
                        "type": "string",
                        "index": 1
                    }],
                    "encoding": "utf-8",
                    "hbaseConfig": {
                        "hbase.zookeeper.quorum": "hbase2:2181",
                        "zookeeper.znode.parent": "/hbase"
                    },
                    "rowkeyColumn": [{
                        "name": "c1:id",
                        "type": "string",
                        "index": 0
                    }, {
                        "name": "c1:name",
                        "type": "string",
                        "index": 1
                    }]
                }
            }
        }],
        "setting": {
            "speed": {
                "channel": 1
            },
            "errorLimit": {
                "record": 0,
                "percentage": 0.02
            }
        }
    }
}
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/mysql2hbase.json
```

### 2.5 Oracle导入数据到HDFS

```bash
python /opt/datax/bin/datax.py -r oraclereader -w hdfswriter  # 模板
cd /opt/datax/job
vi oracle2hdfs.json
```

```xml
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "oraclereader", 
                    "parameter": {
                        "column": ["ID","NAME","CREATE_TIME"], 
                        "splitPk": "ID",
                        "where" : "BRAND_NAME is not null",
                        "connection": [
                            {
                                "jdbcUrl": ["jdbc:oracle:thin:@172.17.17.37:1521:ORCL"], 
                                "table": ["pms_product"]
                            }
                        ], 
                        "username": "DATAX",
                        "password": "123456" 
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
                            },
                            {
                                "name":"create_time",
                                "type":"string"
                            }    
                        ], 
                        "defaultFS": "hdfs://hadoop2:9000",
                        "fieldDelimiter": "\t",
                        "fileName": "orac_product.txt",
                        "fileType": "text",
                        "path": "/datax",
                        "writeMode": "append"
                    }
                }
            }
        ], 
        "setting": {
            "speed": {
                "channel": "3"
            }
        }
    }
}
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/oracle2hdfs.json
```

### 2.6 Oracle导入数据到Mysql

```bash
python /opt/datax/bin/datax.py -r oraclereader -w mysqlwriter  # 模板
cd /opt/datax/job
vi oracle2mysql.json
```

```xml
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "oraclereader", 
                    "parameter": {
                        "column": ["ID","NAME","BRAND_NAME","CREATE_TIME"], 
                        "splitPk": "ID",
                        "where" : "BRAND_NAME is not null",
                        "connection": [
                            {
                                "jdbcUrl": ["jdbc:oracle:thin:@172.17.17.37:1521:ORCL"], 
                                "table": ["pms_product"]
                            }
                        ], 
                        "username": "DATAX",
                        "password": "123456" 
                    }
                }, 
                "writer": {
                    "name": "mysqlwriter", 
                    "parameter": {
                        "column": [ 
                            "id",
                            "name",
                            "brand_name",
                            "create_time"
                           
                        ], 
                        "connection": [
                            {
                                "jdbcUrl": "jdbc:mysql://172.17.17.137:3306/mall?useUnicode=true&characterEncoding=UTF-8", 
                                "table": ["pms_product_bak"]
                            }
                        ], 
                        "username": "root", 
                        "password": "root", 
                        "preSql": [], 
                        "session": ["set session sql_mode='ANSI'"], 
                        "writeMode": "update"
                    }
                }
            }
        ], 
        "setting": {
            "speed": {
                "channel": "3"
            }
        }
    }
}
```

建表语句

```sql
-- oracle
create table PMS_PRODUCT (
    id          INTEGER,
    name        VARCHAR2(200),
    brand_name  VARCHAR2(200),
    create_time DATE default sysdate
)

-- mysql
CREATE TABLE `pms_product_bak` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `name` varchar(64) NOT NULL,
    `brand_name` varchar(255) DEFAULT NULL COMMENT '品牌名称',
    `create_time` datetime DEFAULT NULL,
    PRIMARY KEY (`id`)
) 
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/oracle2mysql.json
```

### 2.7 MongoDB导入数据到HDFS

```bash
python /opt/datax/bin/datax.py -r mongodbreader -w hdfswriter   # 模板
cd /opt/datax/job
vi mongdb2hdfs.json
```

```xml
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "mongodbreader", 
                    "parameter": {
                        "address": ["127.0.0.1:27017"], 
                        "collectionName": "test", 
                        "column": [
                            {
                                "name":"name",
                                "type":"string"
                            },
                            {
                                "name":"url",
                                "type":"string"
                            }
                        ], 
                        "dbName": "datax", 
                        "userName": "", 
                        "userPassword": ""
                    }
                }, 
                "writer": {
                    "name": "hdfswriter", 
                    "parameter": {
                        "column": [
                            {
                                "name":"name",
                                "type":"string"
                            },
                            {
                                "name":"url",
                                "type":"string"
                            }
                        ], 
                        "defaultFS": "hdfs://hadoop2:9000",
                        "fieldDelimiter": "\t", 
                        "fileName": "mongo_pms_product.txt",
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
/opt/datax/bin/datax.py /opt/datax/job/mongdb2hdfs.json
```

### 2.8 MongoDB导入数据到Mysql

```bash
python /opt/datax/bin/datax.py -r mongodbreader -w mysqlwriter  # 模板
cd /opt/datax/job
vi mongodb2mysql.json
```

```xml
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "mongodbreader", 
                    "parameter": {
                        "address": ["127.0.0.1:27017"], 
                        "collectionName": "test", 
                        "column": [
                            {
                                "name":"name",
                                "type":"string"
                            },
                            {
                                "name":"url",
                                "type":"string"
                            }
                        ], 
                        "dbName": "datax", 
                        "userName": "", 
                        "userPassword": ""
                    }
                }, 
                "writer": {
                    "name": "mysqlwriter", 
                    "parameter": {
                        "column": [
                            "name",
                            "url"
                        ], 
                        "connection": [
                            {
                            "jdbcUrl": "jdbc:mysql://172.17.17.137:3306/mall",
                            "table": ["test"]
                            }
                        ],
                        "username": "root",
                        "password": "root",
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
/opt/datax/bin/datax.py /opt/datax/job/mongodb2mysql.json
```

### 2.9 SQLServer导入数据到HDFS


```bash
python /opt/datax/bin/datax.py -r sqlserverreader -w hdfswriter  # 模板
cd /opt/datax/job
vi sqlserver2hdfs.json
```

```xml
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "sqlserverreader", 
                    "parameter": {
                        "column": [
                            "id", 
                            "name",
                            "create_time"
                        ],  
                        "connection": [
                            {
                                "jdbcUrl": ["jdbc:sqlserver://127.0.0.1:1433;DatabaseName=datax"], 
                                "table": ["pms_product"]
                            }
                        ], 
                        "password": "Pass@w0rd", 
                        "username": "SA"
                    }
                }, 
                "writer": {
                    "name": "hdfswriter", 
                    "parameter": {
                        "column": [
                            {
                                "name":"id",
                                "type":"string"
                            },
                            {
                                "name":"name",
                                "type":"string"
                            },
                            {
                                "name":"create_time",
                                "type":"string"
                            }
                        ], 
                        "defaultFS": "hdfs://hadoop2:9000",
                        "fieldDelimiter": "\t", 
                        "fileName": "mssql_pms_product.txt",
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

建表语句

```sql
-- sqlserver
create table pms_product (
    id int,
    name varchar(100),   
    brand_name varchar(100),  
    create_time datetime
)

-- mysql
CREATE TABLE `pms_product_bak` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `name` varchar(64) NOT NULL,
    `brand_name` varchar(255) DEFAULT NULL COMMENT '品牌名称',
    `create_time` datetime DEFAULT NULL,
    PRIMARY KEY (`id`)
) 
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/sqlserver2hdfs.json
```

### 2.10 SQLServer导入到Mysql

```bash
python /opt/datax/bin/datax.py -r sqlserverreader -w mysqlwriter  # 模板
cd /opt/datax/job
vi sqlserver2mysql.json
```

```xml
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "sqlserverreader", 
                    "parameter": {
                        "column": ["id","name","create_time"], 
                        "splitPk": "id",
                        "where" : "create_time is not null", 
                        "connection": [
                            {
                                "jdbcUrl": ["jdbc:sqlserver://127.0.0.1:1433;DatabaseName=datax"], 
                                "table": ["pms_product"]
                            }
                        ], 
                        "password": "Pass@w0rd", 
                        "username": "SA"
                    }
                }, 
                "writer": {
                    "name": "mysqlwriter", 
                    "parameter": {
                        "column": [ 
                            "id",
                            "name",
                            "create_time"
                           
                        ], 
                        "connection": [
                            {
                                "jdbcUrl": "jdbc:mysql://172.17.17.137:3306/mall?useUnicode=true&characterEncoding=UTF-8", 
                                "table": ["pms_product_bak"]
                            }
                        ], 
                        "username": "root", 
                        "password": "root", 
                        "preSql": [], 
                        "session": ["set session sql_mode='ANSI'"], 
                        "writeMode": "update"
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
/opt/datax/bin/datax.py /opt/datax/job/sqlserver2mysql.json
```

### 2.11 PostgreSQL导入数据到HDFS

```bash
python /opt/datax/bin/datax.py -r postgresqlreader -w hdfswriter  # 模板
cd /opt/datax/job
vi postgresql2hdfs.json
```

```xml
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "postgresqlreader", 
                    "parameter": {
                        "connection": [
                            {
                                "jdbcUrl": ["jdbc:postgresql://172.17.17.29:5432/datax"], 
                                "querySql": [
                                    "select id,name,create_time from pms_product where id > 5;"
                                ]
                            }
                        ], 
                        "password": "postgres", 
                        "username": "postgres"
                    }
                }, 
                "writer": {
                    "name": "hdfswriter", 
                    "parameter": {
                        "column": [
                            {
                                "name":"id",
                                "type":"string"
                            },
                            {
                                "name":"name",
                                "type":"string"
                            },
                            {
                                "name":"create_time",
                                "type":"string"
                            }
                        ], 
                        "defaultFS": "hdfs://hadoop2:9000",
                        "fieldDelimiter": "\t", 
                        "fileName": "postgre_pms_product.txt",
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

建表语句

```sql
-- postgresql
create table pms_product (
    id INT8 NOT NULL,
    name varchar(100),   
    brand_name varchar(100),  
    create_time timestamptz,
    PRIMARY KEY (id)
)
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/postgresql2hdfs.json
```

### 2.12 PostgreSQL导入数据到PostgreSQL

```bash
python /opt/datax/bin/datax.py -r postgresqlreader -w postgresqlwriter  # 模板
cd /opt/datax/job
vi postgresql2postgresql.json
```

```xml
{
    "job": {
        "content": [{
            "reader": {
                "name": "postgresqlreader",
                "parameter": {
                    "connection": [{
                        "jdbcUrl": ["jdbc:postgresql://172.17.17.39:5432/vjsp_digit"],
                        "querySql": [
                            "SELECT t1.id,t4.pcode,t3.name,to_timestamp(t1.begin_time,'yyyy-MM-dd hh24:mi:ss') as begin_time,num,TO_CHAR(TO_DATE(t1.begin_time, 'yyyy/MM/dd'), 'yyyy-MM') AS work_date,TO_CHAR(TO_DATE(t1.begin_time, 'yyyy/MM/dd'), 'yyyy') AS year FROM team_task_work_time t1 INNER JOIN team_task t2 ON t1.task_code = t2.code LEFT JOIN team_member t3 ON t1.member_code = t3.code INNER JOIN team_project t4 ON t2.project_code = t4.code AND t4.deleted = 0;"
                        ]
                    }],
                    "password": "postgres",
                    "username": "postgres"
                }
            },
            "writer": {
                "name": "postgresqlwriter",
                "parameter": {
                    "column": [
                        "ts_id",
                        "project_code",
                        "user_name",
                        "createtime",
                        "count",
                        "work_date",
                        "year"
                    ],
                    "connection": [{
                        "jdbcUrl": "jdbc:postgresql://172.17.17.116:5432/VJSP40011321",
                        "table": ["VJSP_IOT_PROJECT_WORK"]
                    }],
                    "preSql": ["delete from @table"],
                    "password": "Vjsp@2022",
                    "username": "postgres"
                }
            }
        }],
        "setting": {
            "speed": {
                "channel": "10"
            }
        }
    }
}
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/postgresql2mysql.json
```

### 2.13 Hbase导入数据到HDFS

```bash
python /opt/datax/bin/datax.py -r hbase11xreader -w hdfswriter  # 模板
cd /opt/datax/job
vi hbase2hdfs.json
```

```xml
{
    "job": {
        "content": [{
            "reader": {
                "name": "hbase11xreader",
                "parameter": {
                    "mode": "normal",
                    "table": "test",
                    "column": [{
                        "name": "c1:id",
                        "type": "string"
                    }, {
                        "name": "c1:name",
                        "type": "string"
                    }],
                    "encoding": "utf-8",
                    "hbaseConfig": {
                        "hbase.zookeeper.quorum": "hbase2:2181",
                        "zookeeper.znode.parent": "/hbase"
                    }
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
                    "defaultFS": "hdfs://hadoop2:9000",
                    "fieldDelimiter": "\t", 
                    "fileName": "hbase_pms_product.txt",
                    "fileType": "text", 
                    "path": "/datax", 
                    "writeMode": "append"
                }   
            } 
        }],
        "setting": {
            "speed": {
                "channel": 1
            },
            "errorLimit": {
                "record": 0,
                "percentage": 0.02
            }
        }
    }
}
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/hbase2hdfs.json
```

### 2.14 Hbase导入数据到Mysql

```bash
python /opt/datax/bin/datax.py -r hbase11xreader -w mysqlwriter  # 模板
cd /opt/datax/job
vi hbase2mysql.json
```

```xml
{
    "job": {
        "content": [{
            "reader": {
                "name": "hbase11xreader",
                "parameter": {
                    "mode": "normal",
                    "table": "test",
                    "column": [{
                        "name": "c1:id",
                        "type": "string"
                    }, {
                        "name": "c1:name",
                        "type": "string"
                    }],
                    "encoding": "utf-8",
                    "hbaseConfig": {
                        "hbase.zookeeper.quorum": "hbase2:2181",
                        "zookeeper.znode.parent": "/hbase"
                    }
                }
            },
            "writer": {
                "name": "mysqlwriter",
                "parameter": {
                    "column": ["id", "name"],
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
        }],
        "setting": {
            "speed": {
                "channel": 1
            },
            "errorLimit": {
                "record": 0,
                "percentage": 0.02
            }
        }
    }
}
```

执行

```bash
/opt/datax/bin/datax.py /opt/datax/job/hbase2mysql.json
```