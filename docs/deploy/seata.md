
# Seata 1.4.2

Seata是一款由阿里巴巴开源的一站式分布式事务解决方案中间件，以高效并且对业务0侵入的方式，解决分布式系统中的数据一致性问题。

- https://github.com/apache/incubator-seata/releases/download/v1.4.2/seata-server-1.4.2.tar.gz
- https://github.com/apache/incubator-seata/archive/refs/tags/v1.4.2.tar.gz


核心概念组件：
- Transaction Coordinator（TC）：事务的协调者seata-server，用于接收我们的事务的注册，提交和回滚。
- Resource Manager（RM）：具体的事务资源，每一个 RM 都会作为一个分支事务注册在 TC。
- Transaction Manager（TM）：事务的发起者。用来告诉 TC，全局事务的开始，提交，回滚。


Seata提供了四种分布式事务模式：
1. `XA模式`：这是一种强一致性的事务解决方案，基于数据库隔离，无代码侵入。它在一阶段不提交事务，而是注册分支事务到事务协调者（TC），并在二阶段根据TC的指令提交或回滚事务。XA模式的优点是事务的强一致性，满足ACID原则，且常用数据库都支持，实现简单。然而，它的缺点是一阶段锁定数据库资源，等待二阶段才释放，性能较差，且`依赖关系型数据库`实现事务。
2. `AT模式`：这是Seata的`默认`事务模式，基于全局锁隔离，`无代码侵入`。在一阶段提交事务前，会记录undolog日志。在二阶段，如果TC通知回滚，则根据undolog回滚；通知提交时，则删除undolog日志。框架的快照性能会影响性能，但是`比XA模式要好`。
3. `TCC模式`：这是一种通过将事务分解为三个阶段（Try、Confirm、Cancel）来实现事务的可靠性和一致性的模式。它要求参与者实现相应的接口和逻辑，确保Try和Cancel操作是幂等的，以处理重试和故障恢复情况。TCC模式的优点是性能最好，不需要依赖关系型数据库，适合`非关系型数据库`的场景，但代码`入侵度较高`。
4. `SAGA模式`：这是一种长事务解决方案，用于处理跨多个服务的业务流程。在SAGA模式中，业务流程中每个参与者都提交本地事务，当出现某一个参与者失败时则补偿前面已经成功的参与者。SAGA模式的优点是能够处理长事务场景，但缺点是代码入侵度较高。

## 1. 安装

### 1.1 上传解压

```bash
cd /opt/software
tar -zxvf seata-server-1.4.2.tar.gz
mv /opt/seata/seata-server-1.4.2 /opt/seata-server-1.4.2 
rm -rf /opt/seata
```

### 1.2 修改file.conf文件

```bash
vi /opt/seata-server-1.4.2/conf/file.conf
```

```conf
store {
  mode = "db"
  db {
    datasource = "druid"
    dbType = "postgresql"
    driverClassName = "org.postgresql.Driver"
    url = "jdbc:postgresql://127.0.0.1:5432/seata"
    user = "seata"
    password = "123456"
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

### 1.3 修改registry.conf文件

```bash
vi /opt/seata-server-1.4.2/conf/registry.conf
```

```conf
registry {
  type = "nacos"
  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    group = "SEATA_GROUP"
    namespace = "628ea892-099a-4d3b-89ec-b8a32a03c530"
    cluster = "default"
    username = "nacos"
    password = "nacos"
  }
}

config {
  type = "nacos"
  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = "628ea892-099a-4d3b-89ec-b8a32a03c530"
    group = "SEATA_GROUP"
    username = "nacos"
    password = "nacos"
  }
}
```

### 1.4 初始化配置文件到nacos

```bash
cd /opt/software
tar -zxvf incubator-seata-1.4.2.tar.gz -C /opt
cd /opt/incubator-seata-1.4.2/script/config-center/
```

```bash
vi config.txt
```

```conf
service.vgroupMapping.test_tx_service_group=default
store.mode=db
store.db.datasource=druid
store.db.dbType=postgresql
store.db.driverClassName=org.postgresql.Driver
store.db.url=jdbc:postgresql://172.17.17.57:5432/ec_oauth_center
store.db.user=ec_oauth_center
store.db.password=Fz52cACAlPJFHHOP
store.db.minConn=5
store.db.maxConn=100
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.lockTable=lock_table
store.db.queryLimit=100
store.db.maxWait=5000
```


执行初始化

```bash
cd /opt/incubator-seata-1.4.2/script/config-center/nacos
sh nacos-config.sh -h 127.0.0.1 -p 8848 -t 628ea892-099a-4d3b-89ec-b8a32a03c530 -g SEATA_GROUP -u nacos -w nacos
```


### 1.5 配置server数据库

进入`/opt/incubator-seata-1.4.2/script/server/db`文件夹，选择相应的数据库新建（数据库名和上述配置需要对应上，默认seata）

### 1.6 配置客户端数据库

> 客户端的表需要手动创建

### 1.7 启动服务


启动seata-server

