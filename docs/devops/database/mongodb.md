# MongoDB 4.4.6

## 1. 安装

1. 下载

- MongoDB Compass 下载：https://www.mongodb.com/try/download/compass
- Robo 3T 下载：https://robomongo.org/download
- mongodb-linux-x86_64 https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.4.6.tgz


2. 解压

```bash
mkdir -p /mydata/mongodb 
tar -xvf mongodb-linux-x86_64-rhel70-4.4.6.tgz -C /mydata/
mv /mydata/mongodb-linux-x86_64-rhel70-4.4.6 /usr/local/mongodb
```

3. 配置

```bash
# 数据存储目录
mkdir -p /mydata/mongodb/data/db
# 日志存储目录
mkdir -p /mydata/mongodb/log
# 新建并修改配置文件
vi /mydata/mongodb/mongod.conf
```

```conf
systemLog:
    # MongoDB发送所有日志输出的目标指定为文件
    destination: file
    # mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mydata/mongodb/log/mongod.log"
    # 当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    # mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mydata/mongodb/data/db"
    journal:
        # 启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    # 启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
net:
    # 服务实例绑定的IP，默认是localhost
    bindIp: localhost,192.168.3.200
    # 绑定的端口，默认是27017
    port: 27017
```

4. 启动关闭

```bash
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/mongod.conf

# 关闭
mongo --port 27017
# 切换到admin库
use admin
# 关闭服务
db.shutdownServer()

# 删除lock文件
rm -f /mydata/mongodb/data/db/*.lock
# 修复数据
/usr/local/mongodb/bin/mongod --repair --dbpath=/mydata/mongodb/data/db
```

## 2. 命令

```bash
# 数据操作
use test        # 选择切换数据库
show dbs        # 查看有权限查看的所有的数据库命令
db.dropDatabase()   # 删除数据库
db.shutdownServer() # 关闭服务  kill关闭会导致文件损坏，使用repair修复
db.createCollection("comment") # 创建集合
show collections               # 查询所有集合
db.collection.drop()           # 删除集合

# 用户角色认证
db.createUser({user:"myroot",pwd:"123456",roles:["root"]})  # 创建超管
db.createUser({user:"myadmin",pwd:"123456",roles:[{role:"userAdminAnyDatabase",db:"admin"}]})   # 创建管理员
db.createUser({user: "xzh", pwd: "123456", roles: [{ role: "readWrite", db:"articledb" }]}) # 创建普通用户
db.system.users.find()                      # 查看已经创建了的用户的情况
db.dropUser("myadmin")                      # 删除用户
db.changeUserPassword("myroot", "123456")   # 修改密码
db.auth("myroot","12345")                   # 用户认证

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

## 3. 高可用

### 3.1 副本集

1. 主节点

```conf
systemLog:
    # MongoDB发送所有日志输出的目标指定为文件
    destination: file
    # mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mydata/mongodb/log/mongod.log"
    # 当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    # mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mydata/mongodb/data/db"
    journal:
        # 启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    # 启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
net:
    # 服务实例绑定的IP，默认是localhost
    bindIp: localhost,192.168.3.200
    # 绑定的端口，默认是27017
    port: 27017
replication:
    #副本集的名称
    replSetName: myrs
```

启动主节点
```bash
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/mongod.conf
```

2. 副本节点

```conf
systemLog:
    # MongoDB发送所有日志输出的目标指定为文件
    destination: file
    # mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mydata/mongodb/log/mongod.log"
    # 当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    # mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mydata/mongodb/data/db"
    journal:
        # 启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    # 启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
net:
    # 服务实例绑定的IP，默认是localhost
    bindIp: localhost,192.168.3.201
    # 绑定的端口，默认是27017
    port: 27018
replication:
    #副本集的名称
    replSetName: myrs
```

启动副本节点
```bash
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/mongod.conf
```

3. 仲裁节点

```conf
systemLog:
    # MongoDB发送所有日志输出的目标指定为文件
    destination: file
    # mongod或mongos应向其发送所有诊断日志记录信息的日志文件的路径
    path: "/mydata/mongodb/log/mongod.log"
    # 当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾。
    logAppend: true
storage:
    # mongod实例存储其数据的目录。storage.dbPath设置仅适用于mongod。
    dbPath: "/mydata/mongodb/data/db"
    journal:
        # 启用或禁用持久性日志以确保数据文件保持有效和可恢复。
        enabled: true
processManagement:
    # 启用在后台运行mongos或mongod进程的守护进程模式。
    fork: true
net:
    # 服务实例绑定的IP，默认是localhost
    bindIp: localhost,192.168.3.202
    # 绑定的端口，默认是27017
    port: 27019
replication:
    #副本集的名称
    replSetName: myrs
```

启动仲裁节点
```bash
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/mongod.conf
```

4. 初始化配置

```bash
/usr/local/mongodb/bin/mongo --host=192.168.3.200 --port=27017
rs.initiate()     # 备初始化新的副本集

rs.conf()     # 查看副本集配置
rs.status()   # 查看节点运行状态
```

副本集配置的查看命令，本质是查询的是system.replset的表中的数据
```bash
use local
show collections
db.system.replset.find()
```

添加节点
```bash
rs.add("192.168.3.201:27018")       # 添加副本从节点
rs.addArb("192.168.3.202:27019")    # 添加仲裁从节点

rs.status()
rs.stepDown(600)                    # 主节点下线
```

5. 副本节点配置

加读的权限
```bash
rs.slaveOk()       # 该命令是 db.getMongo().setSlaveOk() 的简化命令
rs.slaveOk(false)  # 取消从节点读取操作 
```

### 3.2 分片集群

#### 3.2.1 存储节点副本集1

1. 创建目录

```bash
#-----------myshardrs01
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs01_27017/data/db  \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs01_27017/log      \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs01_27018/data/db  \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs01_27018/log      \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs01_27019/data/db  \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs01_27019/log      
```

2. 主节点

```bash
vim /mydata/mongodb/sharded_cluster/myshardrs01_27017/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/myshardrs01_27017/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/mydata/mongodb/sharded_cluster/myshardrs01_27017/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.200
    port: 27017
replication:
    replSetName: myshardrs01
sharding:
    #分片角色
    clusterRole: shardsvr	
```

3. 从节点

```bash
vim /mydata/mongodb/sharded_cluster/myshardrs01_27018/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/myshardrs01_27018/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/mydata/mongodb/sharded_cluster/myshardrs01_27018/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.200
    port: 27018
replication:
    replSetName: myshardrs01
sharding:
    clusterRole: shardsvr	
```


4. 仲裁节点

```bash
vim /mydata/mongodb/sharded_cluster/myshardrs01_27019/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/myshardrs01_27019/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/mydata/mongodb/sharded_cluster/myshardrs01_27019/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.200
    port: 27019
replication:
    replSetName: myshardrs01
sharding:
    clusterRole: shardsvr	
```

5. 启动

```bash
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/sharded_cluster/myshardrs01_27017/mongod.conf
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/sharded_cluster/myshardrs01_27018/mongod.conf
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/sharded_cluster/myshardrs01_27019/mongod.conf
ps -ef |grep mongod
```

6. 初始化

```bash
/usr/local/mongodb/bin/mongo --host 192.168.3.200 --port 27017
rs.initiate()
rs.status()
rs.conf()
rs.add("192.168.3.200:27018")     # 添加副本节点
rs.addArb("192.168.3.200:27019")  # 添加仲裁节点
rs.conf()
```

#### 3.2.2 存储节点副本集2

1. 创建目录

```bash
#-----------myshardrs02
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs02_37017/data/db  \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs02_37017/log      \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs02_37018/data/db  \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs02_37018/log      \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs02_37019/data/db  \ &
mkdir -p /mydata/mongodb/sharded_cluster/myshardrs02_37019/log      
```

2. 主节点

```bash
vim /mydata/mongodb/sharded_cluster/myshardrs02_37017/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/myshardrs02_37017/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/mydata/mongodb/sharded_cluster/myshardrs02_37017/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.200
    port: 37017
replication:
    replSetName: myshardrs02
sharding:
    clusterRole: shardsvr	
```

3. 从节点

```bash
vim /mydata/mongodb/sharded_cluster/myshardrs02_37018/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/myshardrs02_37018/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/mydata/mongodb/sharded_cluster/myshardrs02_37018/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.200
    port: 37018
replication:
    replSetName: myshardrs02
sharding:
    clusterRole: shardsvr	
```


4. 仲裁节点

```bash
vim /mydata/mongodb/sharded_cluster/myshardrs02_37019/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/myshardrs02_37019/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/mydata/mongodb/sharded_cluster/myshardrs02_37019/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.200
    port: 37019
replication:
    replSetName: myshardrs02
sharding:
    clusterRole: shardsvr	
```

5. 启动

```bash
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/sharded_cluster/myshardrs02_37017/mongod.conf
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/sharded_cluster/myshardrs02_37018/mongod.conf
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/sharded_cluster/myshardrs02_37019/mongod.conf
ps -ef |grep mongod
```

6. 初始化

```bash
/usr/local/mongodb/bin/mongo --host 192.168.3.200 --port 37017
rs.initiate()
rs.status()
rs.conf()
rs.add("192.168.3.200:37018")     # 添加副本节点
rs.addArb("192.168.3.200:37019")  # 添加仲裁节点
rs.conf()
```

#### 3.2.3 配置节点副本集

1. 创建目录

```bash
#-----------configrs
mkdir -p /mydata/mongodb/sharded_cluster/myconfigrs_27019/log \ &
mkdir -p /mydata/mongodb/sharded_cluster/myconfigrs_27019/data/db \ &
mkdir -p /mydata/mongodb/sharded_cluster/myconfigrs_27119/log \ &
mkdir -p /mydata/mongodb/sharded_cluster/myconfigrs_27119/data/db \ &
mkdir -p /mydata/mongodb/sharded_cluster/myconfigrs_27219/log \ &
mkdir -p /mydata/mongodb/sharded_cluster/myconfigrs_27219/data/db
```

2. 主配置

```bash
vim /mydata/mongodb/sharded_cluster/myconfigrs_27019/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/myconfigrs_27019/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/mydata/mongodb/sharded_cluster/myconfigrs_27019/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.201
    port: 27019
replication:
    replSetName: myconfigrs
sharding:
    clusterRole: configsvr
```

3. 从配置1

```bash
vim /mydata/mongodb/sharded_cluster/myconfigrs_27119/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/myconfigrs_27119/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/mydata/mongodb/sharded_cluster/myconfigrs_27119/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.201
    port: 27119
replication:
    replSetName: myconfigrs
sharding:
    clusterRole: configsvr
```


4. 从配置2

```bash
vim /mydata/mongodb/sharded_cluster/myconfigrs_27219/mongod.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/myconfigrs_27219/log/mongod.log"
    logAppend: true
storage:
    dbPath: "/mydata/mongodb/sharded_cluster/myconfigrs_27219/data/db"
    journal:
        enabled: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.201
    port: 27219
replication:
    replSetName: myconfigrs
sharding:
    clusterRole: configsvr
```

5. 启动

```bash
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/sharded_cluster/myconfigrs_27019/mongod.conf
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/sharded_cluster/myconfigrs_27119/mongod.conf
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/sharded_cluster/myconfigrs_27219/mongod.conf
ps -ef |grep mongod
```

6. 初始化

```bash
/usr/local/mongodb/bin/mongo --host 192.168.3.201 --port 27019
rs.initiate()
rs.status()
rs.conf()
rs.add("192.168.3.201:27119")  # 添加副本节点1
rs.add("192.168.3.201:27219")  # 添加副本节点2
rs.conf()
```

#### 3.2.4 路由节点

1. 创建目录

```bash
#-----------mongos
mkdir -p /mydata/mongodb/sharded_cluster/mymongos_27017/log  \ &
mkdir -p /mydata/mongodb/sharded_cluster/mymongos_27117/log
```

2. 路由1

```bash
vi /mydata/mongodb/sharded_cluster/mymongos_27017/mongos.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/mymongos_27017/log/mongod.log"
    logAppend: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.201
    port: 27017
sharding:
    configDB: myconfigrs/192.168.3.201:27019,192.168.3.201:27119,192.168.3.201:27219
```

```bash
/usr/local/mongodb/bin/mongos -f /mydata/mongodb/sharded_cluster/mymongos_27017/mongos.conf
```

3. 路由2

```bash
vi /mydata/mongodb/sharded_cluster/mymongos_27117/mongos.conf
```

```conf
systemLog:
    destination: file
    path: "/mydata/mongodb/sharded_cluster/mymongos_27117/log/mongod.log"
    logAppend: true
processManagement:
    fork: true
net:
    bindIp: localhost,192.168.3.201
    port: 27117
sharding:
    configDB: myconfigrs/192.168.3.201:27019,192.168.3.201:27119,192.168.3.201:27219
```

```bash
/usr/local/mongodb/bin/mongos -f /mydata/mongodb/sharded_cluster/mymongos_27117/mongos.conf
```

4. 分片配置

```bash
# 登录客户端
/usr/local/mongodb/bin/mongo --host 192.168.3.201 --port 27017  

# 添加分片
sh.addShard("myshardrs01/192.168.3.200:27017,192.168.3.200:27018,192.168.3.200:27019")    # 添加第一套
sh.addShard("myshardrs02/192.168.3.200:37017,192.168.3.200:37018,192.168.3.200:37019")    # 添加第二套
sh.status()
# 移除分片
use admin
db.runCommand( { removeShard: "myshardrs01" } )
# 开启分片
sh.enableSharding("articledb") 
sh.status()
# 指定分片键
sh.shardCollection("articledb.comment",{"nickname":"hashed"})   # 哈希策略 
sh.status()
sh.shardCollection("articledb.author",{"age":1})                # 范围策略
sh.status()

# 测试哈希规则,从路由上插入的数据，必须包含片键，否则无法插入
use articledb
for(var i=1;i<=1000;i++){db.comment.insert({_id:i+"",nickname:"BoBo"+i})}

# 通过路由节点临时调整默认数据块大小，默认64M
use config
db.settings.save( { _id:"chunksize", value: 1 } )
# 测试范围规则
use articledb 
for(var i=1;i<=20000;i++){db.author.save({"name":"xzhhhhhhhhh"+i,"age":NumberInt(i%120)})}
```

### 3.3 安全认证

#### 3.3.1 角色

- 数据库用户角色：read、readWrite;
- 所有数据库用户角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
- 数据库管理角色：dbAdmin、dbOwner、userAdmin；
- 集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；
- 备份恢复角色：backup、restore；
- 超级用户角色：root
- 内部角色：system

```lua
read 可以读取指定数据库中任何数据。
readWrite 可以读写指定数据库中任何数据，包括创建、重命名、删除集合。
readAnyDatabase 可以读取所有数据库中任何数据(除了数据库config和local之外)。
readWriteAnyDatabase 可以读写所有数据库中任何数据(除了数据库config和local之外)。
userAdminAnyDatabase 可以在指定数据库创建和修改用户(除了数据库config和local之外)。
dbAdminAnyDatabase 可以读取任何数据库以及对数据库进行清理、修改、压缩、获取统计信息、执行检查等操作(除了数据库config和local之外)。
dbAdmin 可以读取指定数据库以及对数据库进行清理、修改、压缩、获取统计信息、执行检查等操作。
userAdmin 可以在指定数据库创建和修改用户。
clusterAdmin 可以对整个集群或数据库系统进行管理操作。
backup 备份MongoDB数据最小的权限。
restore 从备份文件中还原恢复MongoDB数据(除了system.profile集合)的权限。
root 超级账号，超级权限
```

#### 3.3.2 单机认证

参数方式
```bash
/usr/local/mongodb/bin/mongod -f /mydata/mongodb/mongod.conf --auth
```

配置方式
```bash
vi /mydata/mongodb/mongod.conf
```

```conf
security:
    #开启授权认证
    authorization: enabled
```



#### 3.3.3 副本集认证

开启认证之前，创建超管用户，也可以通过localhost创建超管用户

```bash
use admin
db.createUser({user:"myroot",pwd:"123456",roles:["root"]})
```

创建认证文件

```bash
openssl rand -base64 90 -out /mydata/mongodb/mongo.keyfile
scp /mydata/mongodb/mongo.keyfile root@192.168.3.201:/mydata/mongodb/mongo.keyfile
chmod 400 /mydata/mongodb/mongo.keyfile   # 修改权限
vi /mydata/mongodb/mongod.conf
```

```conf
security:
    # KeyFile鉴权文件
    keyFile: /mydata/mongodb/mongo.keyfile
    # 开启认证方式运行
    authorization: enabled
```

在主节点上添加普通账号
```bash
#先用管理员账号登录再切换到admin库
use admin
#管理员账号认证
db.auth("myroot","123456")
#切换到要认证的库
use articledb
#添加普通用户
db.createUser({user: "xzh", pwd: "123456", roles: ["readWrite"]})
```



