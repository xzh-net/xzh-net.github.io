
# 分布式事务Seata

## 1. 分布式事务解决方案
* 2PC
  - XA方案
  - Seata方案（AT、MT）
* TCC
  - Hmily方案 
* 本地消息表（ebay）
* 可靠消息最终一致性
  - 非事务型事务 activemq、rabbitmq、kafka
  - 事务型消息 rocketmq
* 最大努力通知


## 2. Seata介绍

整体事务逻辑是基于两阶段提交的模型，核心概念包括以下3个角色：
- TM：事务的发起者。用来告诉TC，全局事务的开始，提交，回滚。
- RM：具体的事务资源，每一个RM都会作为一个分支事务注册在 TC。
- TC：事务的协调者seata-server，用于接收我们的事务的注册，提交和回滚。

### 2.1. AT模式

该模式适合的场景：
* 基于支持本地ACID事务的关系型数据库。
* Java应用，通过JDBC访问数据库。

### 2.2. MT模式

该模式逻辑类似TCC，需要自定义实现prepare、commit和rollback的逻辑，适合非关系型数据库的场景

## 3. 部署

### 3.1 将seata配置文件加载到nacos

下载 https://seata.io/zh-cn/blog/download.html
* https://github.com/seata/seata/releases/download/v1.4.0/seata-server-1.4.0.zip
* https://github.com/seata/seata/archive/v1.4.0.zip


解压源码压缩包，进入script\config-center文件夹，打开config.txt文件
```conf
service.vgroupMapping.test_tx_service_group=default
store.mode=db
store.db.datasource=druid
store.db.dbType=postgresql
store.db.driverClassName=org.postgresql.Driver
store.db.url=jdbc:postgresql://172.17.17.29:5432/seata
store.db.user=postgres
store.db.password=postgres
store.db.minConn=5
store.db.maxConn=30
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.queryLimit=100
store.db.lockTable=lock_table
store.db.maxWait=5000
```

源码包根目录下script\config-center\nacos下执行sh文件
```bash
sh nacos-config.sh -h 127.0.0.1 -p 8848 -t 1207ed15-5658-48db-a35a-8e3725930070 -g SEATA_GROUP -u nacos -w nacos
-h  # nacos’s host
-p  # nacos’s 端口
-t  # nacos namespace命名空间id
-u  # nacos用户名
-w  # nacos密码
```


### 3.2 配置数据库

进入script\server\db文件夹，选择相应的数据库新建（数据库名和上述配置需要对应上，默认seata）
```sql
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS public.global_table
(
    xid                       VARCHAR(128) NOT NULL,
    transaction_id            BIGINT,
    status                    SMALLINT     NOT NULL,
    application_id            VARCHAR(32),
    transaction_service_group VARCHAR(32),
    transaction_name          VARCHAR(128),
    timeout                   INT,
    begin_time                BIGINT,
    application_data          VARCHAR(2000),
    gmt_create                TIMESTAMP(0),
    gmt_modified              TIMESTAMP(0),
    CONSTRAINT pk_global_table PRIMARY KEY (xid)
);

CREATE INDEX idx_gmt_modified_status ON public.global_table (gmt_modified, status);
CREATE INDEX idx_transaction_id ON public.global_table (transaction_id);

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS public.branch_table
(
    branch_id         BIGINT       NOT NULL,
    xid               VARCHAR(128) NOT NULL,
    transaction_id    BIGINT,
    resource_group_id VARCHAR(32),
    resource_id       VARCHAR(256),
    branch_type       VARCHAR(8),
    status            SMALLINT,
    client_id         VARCHAR(64),
    application_data  VARCHAR(2000),
    gmt_create        TIMESTAMP(6),
    gmt_modified      TIMESTAMP(6),
    CONSTRAINT pk_branch_table PRIMARY KEY (branch_id)
);

CREATE INDEX idx_xid ON public.branch_table (xid);

-- the table to store lock data
CREATE TABLE IF NOT EXISTS public.lock_table
(
    row_key        VARCHAR(128) NOT NULL,
    xid            VARCHAR(128),
    transaction_id BIGINT,
    branch_id      BIGINT       NOT NULL,
    resource_id    VARCHAR(256),
    table_name     VARCHAR(32),
    pk             VARCHAR(36),
    gmt_create     TIMESTAMP(0),
    gmt_modified   TIMESTAMP(0),
    CONSTRAINT pk_lock_table PRIMARY KEY (row_key)
);

CREATE INDEX idx_branch_id ON public.lock_table (branch_id);
```

### 3.3 修改seata-server配置并启动服务

进入conf文件夹，修改registry.conf文件
```conf
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    group = "SEATA_GROUP"
    namespace = "1207ed15-5658-48db-a35a-8e3725930070"
    cluster = "default"
    username = "nacos"
    password = "nacos"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"

  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = "1207ed15-5658-48db-a35a-8e3725930070"
    group = "SEATA_GROUP"
    username = "nacos"
    password = "nacos"
  }
}
```


进入conf文件夹，修改file.conf文件
```conf
## transaction log store, only used in seata-server
store {
  ## store mode: file、db、redis
  mode = "db"
  ## rsa decryption public key
  publicKey = ""
  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "postgresql"
    driverClassName = "org.postgresql.Driver"
    ## if using mysql to store the data, recommend add rewriteBatchedStatements=true in jdbc connection param
    url = "jdbc:postgresql://172.17.17.29:5432/seata"
    user = "postgres"
    password = "vjsp2020"
    minConn = 5
    maxConn = 100
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }
}
```

启动seata-server

