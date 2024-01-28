# MongoDB 4.4.6

- MongoDB Compass 下载：https://www.mongodb.com/try/download/compass
- Robo 3T 下载：https://robomongo.org/download
- mongodb-linux-x86_64 https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.4.6.tgz

## 1. 安装

### 1.1 单机

#### 1.1.1 上传解压

```bash
cd /opt/software/
tar -xvf mongodb-linux-x86_64-rhel70-4.4.6.tgz
mv /opt/software/mongodb-linux-x86_64-rhel70-4.4.6 /usr/local/mongodb
```

#### 1.1.2 修改配置

```bash
mkdir -p /data/mongodb/data/db      # 数据存储目录
mkdir -p /data/mongodb/log          # 日志存储目录
# 新建并修改配置文件
vi /data/mongodb/mongod.conf
```

```conf
systemLog:
    # MongoDB发送所有日志输出的目标指定为文件
    destination: file
    # mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/data/mongodb/log/mongod.log"
    # 当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    # mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/data/mongodb/data/db"
    journal:
        # 启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    # 启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
net:
    # 服务实例绑定的IP，默认是localhost
    bindIp: localhost,192.168.2.201
    # 绑定的端口，默认是27017
    port: 27017
security:
    # 开启授权认证，必须在创建管理员用户以后开启
    authorization: enabled
```

#### 1.1.3 启动服务

```bash
/usr/local/mongodb/bin/mongod -f /data/mongodb/mongod.conf
```

如果异常关闭后无法启动，需要删除lock文件

```bash
rm -f /data/mongodb/data/db/*.lock      # 删除lock文件
/usr/local/mongodb/bin/mongod --repair --dbpath=/data/mongodb/data/db   # 修复数据
```

#### 1.1.4 客户端测试

```bash
/usr/local/mongodb/bin/mongo --port 27017
use admin
db.createUser( {user: "root", pwd: "123456", roles: [ { role: "root", db: "admin" } ] } )       # 创建超管
db.createUser( {user: "admin", pwd: "123456", roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] } )     # 创建管理员
db.createUser( {user: "xzh", pwd: "123456", roles: [ { role: "readWrite", db:"articledb" } ] } )               # 创建普通用户
db.system.users.find()
# 插入数据
use articledb
db.comment.insert({"articleid":"100000","content":"今天天气真好，阳光明媚","userid":"1001","nickname":"Rose","createdatetime":new Date()})
db.comment.find()
```

### 1.2 副本集

| **节点名称** | **IP地址** | **服务名称** | **端口** | **配置文件** |
| :----------: | :----------: | :----------: | :----------: | :----------: |
| 主节点  | node201 192.168.2.201 | mongod  | 27017 | /data/mongodb/replica_sets/myrs_27017/mongod.conf	 |
| 从节点  | node201 192.168.2.201 | mongod  | 27018 | /data/mongodb/replica_sets/myrs_27018/mongod.conf	 |
| 仲裁节点  | node201 192.168.2.201 | mongod  | 27019 | /data/mongodb/replica_sets/myrs_27019/mongod.conf |


#### 1.2.1 创建主节点

```bash
mkdir -p /data/mongodb/replica_sets/myrs_27017/log
mkdir -p /data/mongodb/replica_sets/myrs_27017/data/db

vim /data/mongodb/replica_sets/myrs_27017/mongod.conf
```

```conf
systemLog:
    # MongoDB发送所有日志输出的目标指定为文件
    destination: file
    # mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/data/mongodb/replica_sets/myrs_27017/log/mongod.log"
    # 当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    # mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/data/mongodb/replica_sets/myrs_27017/data/db"
    journal:
        # 启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    # 启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
net:
    # 服务实例绑定的IP，默认是localhost
    bindIp: localhost,192.168.2.201
    # 绑定的端口，默认是27017
    port: 27017
replication:
    #副本集的名称
    replSetName: myrs
```

启动主节点

```bash
/usr/local/mongodb/bin/mongod -f /data/mongodb/replica_sets/myrs_27017/mongod.conf
```

#### 1.2.2 创建副本节点

```bash
mkdir -p /data/mongodb/replica_sets/myrs_27018/log
mkdir -p /data/mongodb/replica_sets/myrs_27018/data/db

vim /data/mongodb/replica_sets/myrs_27018/mongod.conf
```

```conf
systemLog:
    # MongoDB发送所有日志输出的目标指定为文件
    destination: file
    # mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/data/mongodb/replica_sets/myrs_27018/log/mongod.log"
    # 当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    # mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/data/mongodb/replica_sets/myrs_27018/data/db"
    journal:
        # 启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    # 启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
net:
    # 服务实例绑定的IP，默认是localhost
    bindIp: localhost,192.168.2.201
    # 绑定的端口，默认是27017
    port: 27018
replication:
    #副本集的名称
    replSetName: myrs
```

启动副本节点

```bash
/usr/local/mongodb/bin/mongod -f /data/mongodb/replica_sets/myrs_27018/mongod.conf
```

#### 1.2.3 创建仲裁节点

```bash
mkdir -p /data/mongodb/replica_sets/myrs_27019/log
mkdir -p /data/mongodb/replica_sets/myrs_27019/data/db

vim /data/mongodb/replica_sets/myrs_27019/mongod.conf
```

```conf
systemLog:
    # MongoDB发送所有日志输出的目标指定为文件
    destination: file
    # mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/data/mongodb/replica_sets/myrs_27019/log/mongod.log"
    # 当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    # mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/data/mongodb/replica_sets/myrs_27019/data/db"
    journal:
        # 启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    # 启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
net:
    # 服务实例绑定的IP，默认是localhost
    bindIp: localhost,192.168.2.201
    # 绑定的端口，默认是27017
    port: 27019
replication:
    #副本集的名称
    replSetName: myrs
```

启动仲裁节点

```bash
/usr/local/mongodb/bin/mongod -f /data/mongodb/replica_sets/myrs_27019/mongod.conf
```

#### 1.2.4 初始化副本集

1. 使用客户端命令连接主节点

```bash
/usr/local/mongodb/bin/mongo --host=192.168.2.201 --port=27017
rs.initiate()       # 初始化新的副本集
rs.conf()           # 查看副本集配置
rs.status()         # 查看节点运行状态
```

副本集配置的查看命令，本质是查询的是system.replset的表中的数据

```bash
use local
show collections
db.system.replset.find()
```

2. 添加从节点

在主节点添加从节点，将其他成员加入到副本集

```bash
rs.add("192.168.2.201:27018")       # 添加副本从节点
rs.addArb("192.168.2.201:27019")    # 添加仲裁从节点
rs.status()
```

#### 1.2.5 客户端测试

1. 登录主节点

```bash
/usr/local/mongodb/bin/mongo --host=192.168.2.201 --port=27017
use articledb
db.comment.insert({"articleid":"100001","content":"我们都是好孩子","userid":"1002","nickname":"xzh","createdatetime":new Date()})
```

2. 登录从节点

```bash
/usr/local/mongodb/bin/mongo --host=192.168.2.201 --port=27018
```

从节点不能读取集合的数据，当前从节点只是一个备份，不是奴隶节点，无法读取数据，也不能写入，需要加读的权限

```bash
rs.slaveOk()       # 允许该节点读取数据，该命令是db.getMongo().setSlaveOk()的简化命令
rs.slaveOk(false)  # 取消从节点读取操作 
```

```bash
use articledb
db.comment.find()
```

### 1.3 分片集群

![](../../assets/_images/deploy/mongodb/image1.png)

| **节点名称** | **IP地址** | **服务名称** | **端口** | **配置文件** |
| :----------: | :----------: | :----------: | :----------: | :----------: |
| 路由节点  | node199 192.168.2.199 | mongos1 | 27017 | /data/mongodb/sharded_cluster/mymongos_27017/mongos.conf |
|   |  | mongos2 | 27117 | /data/mongodb/sharded_cluster/mymongos_27117/mongos.conf |
| 配置节点  | node200 192.168.2.200 | mongod1 | 27019 | /data/mongodb/sharded_cluster/myconfigrs_27019/mongod.conf |
|   |  | mongod2 | 27119 | /data/mongodb/sharded_cluster/myconfigrs_27119/mongod.conf |
|   |  | mongod3 | 27219 | /data/mongodb/sharded_cluster/myconfigrs_27219/mongod.conf |
| 存储节点1  | node201 192.168.2.201 | mongod1 | 27017 | /data/mongodb/sharded_cluster/myshardrs01_27017/mongod.conf |
|   |  | mongod3 | 27018 | /data/mongodb/sharded_cluster/myshardrs01_27018/mongod.conf |
|   |  | mongod3 | 27019 | /data/mongodb/sharded_cluster/myshardrs01_27019/mongod.conf |
| 存储节点2  | node202 192.168.2.202 | mongod1 | 37017 | /data/mongodb/sharded_cluster/myshardrs02_37017/mongod.conf |
|   |  | mongod2 | 37018 | /data/mongodb/sharded_cluster/myshardrs02_37018/mongod.conf |
|   |  | mongod3 | 37019 | /data/mongodb/sharded_cluster/myshardrs02_37019/mongod.conf |


#### 1.3.1 存储节点副本集1

在192.168.2.201执行

1. 创建目录

```bash
#-----------myshardrs01
mkdir -p /data/mongodb/sharded_cluster/myshardrs01_27017/data/db  \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs01_27017/log      \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs01_27018/data/db  \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs01_27018/log      \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs01_27019/data/db  \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs01_27019/log      
```

2. 创建主节点

```bash
vim /data/mongodb/sharded_cluster/myshardrs01_27017/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/myshardrs01_27017/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/data/mongodb/sharded_cluster/myshardrs01_27017/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.201
    port: 27017
replication:
    replSetName: myshardrs01
sharding:
    #分片角色
    clusterRole: shardsvr
```

3. 创建从节点

```bash
vim /data/mongodb/sharded_cluster/myshardrs01_27018/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/myshardrs01_27018/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/data/mongodb/sharded_cluster/myshardrs01_27018/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.201
    port: 27018
replication:
    replSetName: myshardrs01
sharding:
    clusterRole: shardsvr
```


4. 仲裁节点

```bash
vim /data/mongodb/sharded_cluster/myshardrs01_27019/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/myshardrs01_27019/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/data/mongodb/sharded_cluster/myshardrs01_27019/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.201
    port: 27019
replication:
    replSetName: myshardrs01
sharding:
    clusterRole: shardsvr
```

5. 启动服务

```bash
/usr/local/mongodb/bin/mongod -f /data/mongodb/sharded_cluster/myshardrs01_27017/mongod.conf
/usr/local/mongodb/bin/mongod -f /data/mongodb/sharded_cluster/myshardrs01_27018/mongod.conf
/usr/local/mongodb/bin/mongod -f /data/mongodb/sharded_cluster/myshardrs01_27019/mongod.conf
ps -ef |grep mongod
```

6. 初始化

```bash
/usr/local/mongodb/bin/mongo --host 192.168.2.201 --port 27017
rs.initiate()
rs.status()
rs.conf()
rs.add("192.168.2.201:27018")     # 添加副本节点
rs.addArb("192.168.2.201:27019")  # 添加仲裁节点
rs.conf()
```

#### 1.3.2 存储节点副本集2

在192.168.2.202执行

1. 创建目录

```bash
#-----------myshardrs02
mkdir -p /data/mongodb/sharded_cluster/myshardrs02_37017/data/db  \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs02_37017/log      \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs02_37018/data/db  \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs02_37018/log      \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs02_37019/data/db  \ &
mkdir -p /data/mongodb/sharded_cluster/myshardrs02_37019/log      
```

2. 创建主节点

```bash
vim /data/mongodb/sharded_cluster/myshardrs02_37017/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/myshardrs02_37017/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/data/mongodb/sharded_cluster/myshardrs02_37017/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.202
    port: 37017
replication:
    replSetName: myshardrs02
sharding:
    clusterRole: shardsvr
```

3. 创建从节点

```bash
vim /data/mongodb/sharded_cluster/myshardrs02_37018/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/myshardrs02_37018/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/data/mongodb/sharded_cluster/myshardrs02_37018/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.202
    port: 37018
replication:
    replSetName: myshardrs02
sharding:
    clusterRole: shardsvr
```


4. 仲裁节点

```bash
vim /data/mongodb/sharded_cluster/myshardrs02_37019/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/myshardrs02_37019/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/data/mongodb/sharded_cluster/myshardrs02_37019/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.202
    port: 37019
replication:
    replSetName: myshardrs02
sharding:
    clusterRole: shardsvr
```

5. 启动服务

```bash
/usr/local/mongodb/bin/mongod -f /data/mongodb/sharded_cluster/myshardrs02_37017/mongod.conf
/usr/local/mongodb/bin/mongod -f /data/mongodb/sharded_cluster/myshardrs02_37018/mongod.conf
/usr/local/mongodb/bin/mongod -f /data/mongodb/sharded_cluster/myshardrs02_37019/mongod.conf
ps -ef |grep mongod
```

6. 初始化

```bash
/usr/local/mongodb/bin/mongo --host 192.168.2.202 --port 37017
rs.initiate()
rs.status()
rs.conf()
rs.add("192.168.2.202:37018")     # 添加副本节点
rs.addArb("192.168.2.202:37019")  # 添加仲裁节点
rs.conf()
```

#### 1.3.3 配置节点副本集

在192.168.2.200执行

1. 创建目录

```bash
#-----------configrs
mkdir -p /data/mongodb/sharded_cluster/myconfigrs_27019/log \ &
mkdir -p /data/mongodb/sharded_cluster/myconfigrs_27019/data/db \ &
mkdir -p /data/mongodb/sharded_cluster/myconfigrs_27119/log \ &
mkdir -p /data/mongodb/sharded_cluster/myconfigrs_27119/data/db \ &
mkdir -p /data/mongodb/sharded_cluster/myconfigrs_27219/log \ &
mkdir -p /data/mongodb/sharded_cluster/myconfigrs_27219/data/db
```

2. 创建主配置

```bash
vim /data/mongodb/sharded_cluster/myconfigrs_27019/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/myconfigrs_27019/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/data/mongodb/sharded_cluster/myconfigrs_27019/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.200
    port: 27019
replication:
    replSetName: myconfigrs
sharding:
    clusterRole: configsvr
```

3. 创建从配置1

```bash
vim /data/mongodb/sharded_cluster/myconfigrs_27119/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/myconfigrs_27119/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/data/mongodb/sharded_cluster/myconfigrs_27119/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.200
    port: 27119
replication:
    replSetName: myconfigrs
sharding:
    clusterRole: configsvr
```


4. 创建从配置2

```bash
vim /data/mongodb/sharded_cluster/myconfigrs_27219/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/myconfigrs_27219/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/data/mongodb/sharded_cluster/myconfigrs_27219/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.200
    port: 27219
replication:
    replSetName: myconfigrs
sharding:
    clusterRole: configsvr
```

5. 启动服务

```bash
/usr/local/mongodb/bin/mongod -f /data/mongodb/sharded_cluster/myconfigrs_27019/mongod.conf
/usr/local/mongodb/bin/mongod -f /data/mongodb/sharded_cluster/myconfigrs_27119/mongod.conf
/usr/local/mongodb/bin/mongod -f /data/mongodb/sharded_cluster/myconfigrs_27219/mongod.conf
ps -ef |grep mongod
```

6. 初始化

```bash
/usr/local/mongodb/bin/mongo --host 192.168.2.200 --port 27019
rs.initiate()
rs.status()
rs.conf()
rs.add("192.168.2.200:27119")  # 添加副本节点1
rs.add("192.168.2.200:27219")  # 添加副本节点2
rs.conf()
```

#### 1.3.4 路由节点

在192.168.2.199执行

1. 创建目录

```bash
#-----------mongos
mkdir -p /data/mongodb/sharded_cluster/mymongos_27017/log  \ &
mkdir -p /data/mongodb/sharded_cluster/mymongos_27117/log
```

2. 创建路由1

```bash
vi /data/mongodb/sharded_cluster/mymongos_27017/mongos.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/mymongos_27017/log/mongod.log"
    logAppend: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.199
    port: 27017
sharding:
    configDB: myconfigrs/192.168.2.200:27019,192.168.2.200:27119,192.168.2.200:27219
```


3. 创建路由2

```bash
vi /data/mongodb/sharded_cluster/mymongos_27117/mongos.conf
```

```conf
systemLog:
    destination: file
    path: "/data/mongodb/sharded_cluster/mymongos_27117/log/mongod.log"
    logAppend: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.2.199
    port: 27117
sharding:
    configDB: myconfigrs/192.168.2.200:27019,192.168.2.200:27119,192.168.2.200:27219
```

4. 启动服务

```bash
/usr/local/mongodb/bin/mongos -f /data/mongodb/sharded_cluster/mymongos_27017/mongos.conf
/usr/local/mongodb/bin/mongos -f /data/mongodb/sharded_cluster/mymongos_27117/mongos.conf
ps -ef |grep mongos
```


#### 1.3.5 初始化集群分片

```bash
# 登录路由客户端
/usr/local/mongodb/bin/mongo --host 192.168.2.199 --port 27017  

# 添加分片
sh.addShard("myshardrs01/192.168.2.201:27017,192.168.2.201:27018,192.168.2.201:27019")    # 添加副本集1
sh.addShard("myshardrs02/192.168.2.202:37017,192.168.2.202:37018,192.168.2.202:37019")    # 添加副本集2
sh.status()
# 移除分片
use admin
db.runCommand( { removeShard: "myshardrs01" } )
# 开启分片
sh.enableSharding("articledb") 
# 指定分片策略
sh.shardCollection("articledb.comment",{"nickname":"hashed"})   # 哈希策略 
sh.shardCollection("articledb.author",{"age":1})                # 范围策略
sh.status()
```

#### 1.3.6 客户端测试

1. 哈希策略

从路由上插入数据

```bash
# 测试哈希规则，必须包含片键，否则无法插入
/usr/local/mongodb/bin/mongo --host 192.168.2.199 --port 27017  
use articledb
for(var i=1;i<=1000;i++){db.comment.insert({_id:i+"",nickname:"xzh"+i})}
```

分别登陆两个副本集的主节点，统计文档数量

```bash
/usr/local/mongodb/bin/mongo --host 192.168.2.201 --port 27017  
use articledb
db.comment.count()
```

```bash
/usr/local/mongodb/bin/mongo --host 192.168.2.202 --port 37017  
use articledb
db.comment.count()
```

2. 范围策略

从路由上插入数据

```bash
# 测试范围
/usr/local/mongodb/bin/mongo --host 192.168.2.199 --port 27017  
use articledb 
for(var i=1;i<=20000;i++){db.author.save({"name":"xzhhhhhhhhh"+i,"age":NumberInt(i%120)})}
```

分别登陆两个副本集的主节点，统计文档数量

```bash
/usr/local/mongodb/bin/mongo --host 192.168.2.201 --port 27017  
use articledb
db.author.count()
```

```bash
/usr/local/mongodb/bin/mongo --host 192.168.2.202 --port 37017  
use articledb
db.author.count()
```

3. 通过路由节点临时调整默认数据块大小

范围测试中，数据块（chunk）没有填满，默认的数据块尺寸（chunksize）是64M，填满后才会考虑向其他片的数据块填充数据，因此，为了测试，可以将其改小

```bash
use config
db.settings.save( { _id:"chunksize", value: 1 } )
```


## 2 安全认证

### 2.1 单实例

1. 参数方式

```bash
/usr/local/mongodb/bin/mongod -f /data/mongodb/mongod.conf --auth
```

2. 配置方式

```bash
vi /data/mongodb/mongod.conf
```
```conf
security:
    #开启授权认证
    authorization: enabled
```



### 2.2 副本集

1. 开启认证之前，创建超管用户，也可以通过localhost创建超管用户

```bash
use admin
db.createUser({user:"myroot",pwd:"123456",roles:["root"]})
```

2. 创建认证文件

```bash
openssl rand -base64 90 -out /data/mongodb/mongo.keyfile
scp /data/mongodb/mongo.keyfile root@192.168.2.201:/data/mongodb/mongo.keyfile
chmod 400 /data/mongodb/mongo.keyfile   # 修改权限
vi /data/mongodb/mongod.conf
```

```conf
security:
    # KeyFile鉴权文件
    keyFile: /data/mongodb/mongo.keyfile
    # 开启认证方式运行
    authorization: enabled
```

3. 在主节点上添加普通账号

```bash
# 先用管理员账号登录再切换到admin库
use admin
# 管理员账号认证
db.auth("myroot","123456")
# 切换到要认证的库
use articledb
# 添加普通用户
db.createUser({user: "xzh", pwd: "123456", roles: ["readWrite"]})
```

### 2.3 分片集群

## 3. 命令

```bash
mongo --host=127.0.0.1 --port=27017

# 数据操作
use test        # 选择切换数据库
show dbs        # 查看有权限查看的所有的数据库命令
db.dropDatabase()       # 删除数据库
db.shutdownServer()     # 关闭服务  kill关闭会导致文件损坏，使用repair修复
db.createCollection("comment")  # 创建集合
show collections                # 查询所有集合
db.collection.drop()            # 删除集合

# 用户角色认证
db.createUser( {user: "root", pwd: "123456", roles: [ { role: "root", db: "admin" } ] } )       # 创建超管
db.createUser( {user: "admin", pwd: "123456", roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] } )     # 创建管理员
db.createUser( {user: "xzh", pwd: "123456", roles: [ { role: "readWrite", db:"articledb" } ] } )               # 创建普通用户
db.system.users.find()                      # 查看已经创建了的用户的情况
db.dropUser("xzh")                          # 删除用户
db.changeUserPassword("xzh", "123456")      # 修改密码
db.auth("xzh","12345")                      # 用户认证

# 角色
db.runCommand({ rolesInfo: 1 }) # 查询所有角色权限(仅用户自定义角色)
db.runCommand({ rolesInfo: 1, showBuiltinRoles: true }) # 查询所有角色权限(包含内置角色)
db.runCommand({ rolesInfo: "<rolename>" })              # 查询当前数据库中的某角色的权限
db.runCommand({ rolesInfo: { role: "<rolename>", db: "<database>" } } # 查询其它数据库中指定的角色权限

# 插入
db.comment.insert({"articleid":"100000","content":"今天天气真好，阳光明媚","userid":"1001","nickname":"Rose","createdatetime":new Date(),"likenum":NumberInt(10),"state":null})
# 批量插入
db.comment.insertMany([
{"_id":"1","articleid":"100001","content":"我们不应该把清晨浪费在手机上，健康很重要，一杯温水幸福你我他。","userid":"1002","nickname":"相忘于江湖","createdatetime":new Date("2019-08-
05T22:08:15.522Z"),"likenum":NumberInt(1000),"state":"1"},
{"_id":"2","articleid":"100001","content":"我夏天空腹喝凉开水，冬天喝温开水","userid":"1005","nickname":"伊人憔悴","createdatetime":new Date("2019-08-05T23:58:51.485Z"),"likenum":NumberInt(888),"state":"1"},
{"_id":"3","articleid":"100001","content":"我一直喝凉开水，冬天夏天都喝。","userid":"1004","nickname":"杰克船长","createdatetime":new Date("2019-08-06T01:05:06.321Z"),"likenum":NumberInt(666),"state":"1"},
{"_id":"4","articleid":"100001","content":"专家说不能空腹吃饭，影响健康。","userid":"1003","nickname":"凯撒","createdatetime":new Date("2019-08-06T08:18:35.288Z"),"likenum":NumberInt(2000),"state":"1"},
{"_id":"5","articleid":"100001","content":"研究表明，刚烧开的水千万不能喝，因为烫嘴。","userid":"1003","nickname":"凯撒","createdatetime":new Date("2019-08-06T11:01:02.521Z"),"likenum":NumberInt(3000),"state":"1"}
]);

# 查询
db.comment.find();                  # 查询所有数据
db.comment.find({{userid:'1002'})   # 条件查询数据
db.comment.findOne({条件})          # 查询符合条件的第一条记录
db.comment.find({条件}).skip(条数).limit(条数)       # 来读limit取指定数量的数据，跳过skip条(分页)
db.comment.find({userid:"1003"},{userid:1,nickname:1,_id:0})  # 投影查询(不显示所有字段，只显示指定的字段)
db.comment.find().sort({userid:-1,likenum:1})       # 排序 按照sort()，skip()，limit()顺序执行和编写顺序无关
db.comment.find({content:/^专家/})                  # 模糊查询
db.comment.find({userid:{$in:["1003","1004"]}})     # 包含查询
db.comment.find({userid:{$nin:["1003","1004"]}})    # 不包含查询
db.comment.find({$and:[{likenum:{$gte:NumberInt(700)}},{likenum:{$lt:NumberInt(2000)}}]}) # 多条件连接查询
db.comment.find({$or:[ {userid:"1003"} ,{likenum:{$lt:1000} }]})

db.comment.find({ "field" : { $gt: value }})    # 大于: field > value
db.comment.find({ "field" : { $lt: value }})    # 小于: field < value
db.comment.find({ "field" : { $gte: value }})   # 大于等于: field >= value
db.comment.find({ "field" : { $lte: value }})   # 小于等于: field <= value
db.comment.find({ "field" : { $ne: value }})    # 不等于: field != value

# 更新
db.comment.update({_id:"1"},{likenum:NumberInt(1001)})       # 覆盖修改
db.comment.update({_id:"2"},{$set:{likenum:NumberInt(889)}}) # 局部修改，默认只修改第一条数据
db.comment.update({userid:"1003"},{$set:{nickname:"凯撒大帝"}},{multi:true}) # 修改所有符合条件的数据
db.comment.update({_id:"3"},{$inc:{likenum:NumberInt(1)}}) # 修改自增某字段值

# 聚合删除
db.comment.remove() # 删除集合全部数据
db.comment.remove({_id:"1"})) # 按条件删除集合数据
db.comment.count({条件})    # 统计查询
db.comment.aggregate([{$group : {_id : "$by", sum_count : {$sum : 1}}}]) # 聚合 $sum $avg $min $max

# 索引
db.collection.getIndexes() # 查看集合所有索引
db.comment.createIndex({userid:1}，{background: true})  # 创建索引 background：建索引过程会阻塞其它数据库操作，true表示后台创建，默认为false
db.comment.dropIndexes()            # 删除全部索引
db.comment.dropIndex({userid:1})    # 删除索引

# 执行计划
db.comment.find({userid:"1003"}).explain()  # 查看执行计划 stage: "COLLSCAN"表示全集合扫描 "IXSCAN" ,基于索引的扫描

# 集群
db.printShardingStatus()    # 集群的详细信息
sh.isBalancerRunning()      # 查看均衡器是否工作
sh.getBalancerState()       # 查看当前Balancer状态

```