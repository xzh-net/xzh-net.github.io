# Pulsar 2.10.1

Apache Pulsar 是一个云原生企业级的发布订阅（pub-sub）消息系统

- 官方网站：https://pulsar.apache.org
- 下载地址：https://pulsar.apache.org/download

## 1. 安装

### 1.1 单机

#### 1.1.1 上传解压

```bash
cd /opt/software
tar -zxvf apache-pulsar-2.10.1-bin.tar.gz -C /opt
```

#### 1.1.2 启动

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar standalone
```

#### 1.1.3 客户端测试

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-client consume my-topic -s "first-subscription"    # 监听数据
./pulsar-client produce my-topic --messages "hello-pulsar"  # 模拟生产数据
```


### 1.2 集群

#### 1.2.1 集群规划

- node01 ZooKeeper bookie broker
- node02 ZooKeeper bookie broker
- node03 ZooKeeper bookie broker

#### 1.2.2 上传解压

```bash
cd /opt/software
tar -zxvf apache-pulsar-2.10.1-bin.tar.gz -C /opt
```

#### 1.2.3 修改bookkeeper集群配置文件

```bash
cd /opt/apache-pulsar-2.10.1/conf
vim bookkeeper.conf
```

编辑内容

```conf
advertisedAddress=node01.xuzhihao.net
journalDirectory=/opt/apache-pulsar-2.10.1/data/bookkeeper/journal
ledgerDirectories=/opt/apache-pulsar-2.10.1/data/bookkeeper/ledgers
zkServers=node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181
```

#### 1.2.4 修改broker集群的配置文件

```bash
cd /opt/apache-pulsar-2.10.1/conf
vim broker.conf
```

编辑内容

```conf
clusterName=pulsar-cluster
zookeeperServers=node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181
configurationStoreServers=node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181
advertisedAddress=node01.xuzhihao.net
```

#### 1.2.5 分发安装包

```
cd /opt/apache-pulsar-2.10.1
scp -r /opt/apache-pulsar-2.10.1 node02:$PWD
scp -r /opt/apache-pulsar-2.10.1 node03:$PWD
```

#### 1.2.6 修改分发后的broker的地址和bookies地址

1. node02节点执行

```bash
cd /opt/apache-pulsar-2.10.1/conf
vim bookkeeper.conf
# 编辑内容
advertisedAddress=node02.xuzhihao.net
```

```bash
vim broker.conf
# 编辑内容
advertisedAddress=node02.xuzhihao.net
```

2. node03节点执行

```bash
cd /opt/apache-pulsar-2.10.1/conf
vim bookkeeper.conf
# 编辑内容
advertisedAddress=node03.xuzhihao.net
```

```bash
vim broker.conf
# 编辑内容
advertisedAddress=node03.xuzhihao.net
```

#### 1.2.7 启动zk集群

```bash
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh start   # 每个节点分别启动
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh status  # 检测节点状态
```

#### 1.2.8 初始化元数据

```bash
cd /opt/apache-pulsar-2.10.1/bin

./pulsar initialize-cluster-metadata \
--cluster pulsar-cluster \
--zookeeper node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181 \
--configuration-store node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181 \
--web-service-url http://node01.xuzhihao.net:8080,node02.xuzhihao.net:8080,node03.xuzhihao.net:8080 \
--web-service-url-tls https://node01.xuzhihao.net:8443,node02.xuzhihao.net:8443,node03.xuzhihao.net:8443 \
--broker-service-url pulsar://node01.xuzhihao.net:6650,node02.xuzhihao.net:6650,node03.xuzhihao.net:6650 \
--broker-service-url-tls pulsar+ssl://node01.xuzhihao.net::6651,node02.xuzhihao.net:6651,node03.xuzhihao.net:6651

./bookkeeper shell metaformat   # 初始化bookkeeper集群: 若出现提示输入Y/N: 请输入Y
```

#### 1.2.9 集群模式启动

1. 启动bookkeeper

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-daemon start bookie    # 三个节点都需要依次启动
./bookkeeper shell bookiesanity # 验证是否启动
```

2. 启动Broker

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-daemon start broker    # 三个节点都需要依次启动
./pulsar-admin brokers list pulsar-cluster  # 验证是否启动
```

#### 1.2.10 集群模式测试

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-client consume persistent://public/default/test -s "consumer-test"         # 监听数据
./pulsar-client produce persistent://public/default/test --messages "hello-pulsar"  # 模拟生产数据
```

## 2. 可视化监控

下载地址：https://archive.apache.org/dist/pulsar/pulsar-manager/pulsar-manager-0.2.0/apache-pulsar-manager-0.2.0-bin.tar.gz

### 2.1 上传解压

```bash
cd /opt/software
tar -zxvf apache-pulsar-manager-0.2.0-bin.tar.gz -C /opt
```

### 2.2 修改配置

```bash
cd /opt/pulsar-manager/
tar -xvf pulsar-manager.tar
cd pulsar-manager
cp -r ../dist ui
```

### 2.3 启动服务

```bash
/opt/pulsar-manager/pulsar-manager/bin/pulsar-manager

cd /opt/pulsar-manager/pulsar-manager
nohup bin/pulsar-manager >> pulsar-manager.log 2>&1 &   # 后台启动
```

### 2.4 设置管理员

```bash
CSRF_TOKEN=$(curl http://localhost:7750/pulsar-manager/csrf-token)

curl \
    -H "X-XSRF-TOKEN: $CSRF_TOKEN" \
    -H "Cookie: XSRF-TOKEN=$CSRF_TOKEN;" \
    -H 'Content-Type: application/json' \
    -X PUT http://localhost:7750/pulsar-manager/users/superuser \
    -d '{"name": "admin", "password": "12345678", "description": "admin", "email": "username@test.org"}'
```

### 2.5 添加集群

#### 2.5.1 访问地址

http://192.168.2.201:7750/ui/index.html

#### 2.5.2 添加集群

http://192.168.2.201:8080,192.168.2.202:8080,192.168.2.203:8080

#### 2.5.3 Clusters无法显示

```bash
cd /opt/apache-pulsar-2.10.1
bin/pulsar-admin clusters list
bin/pulsar-admin clusters update pulsar-cluster --url http://192.168.2.201:8080
```

重启pulsar-manager重新添加集群

## 3. Admin API

https://pulsar.apache.org/docs/next/admin-api-overview

### 3.1 Clusters集群

```bash
cd /opt/apache-pulsar-2.10.1
./pulsar-admin clusters list                # 获取所有集群
./pulsar-admin clusters create cluster-1 --url http://node01:8080 --broker-url pulsar://node01:6650   # 创建集群

# 集群初始化
./pulsar initialize-cluster-metadata \
  --cluster cluster-1 \
  --configuration-metadata-store zk:node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181/pulsar-cluster-1 \
  --metadata-store zk:node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181/pulsar-cluster-1 \
  --web-service-url http://node01.xuzhihao.net:8080,node02.xuzhihao.net:8080,node03.xuzhihao.net:8080 \
  --web-service-url-tls https://node01.xuzhihao.net:8443,node02.xuzhihao.net:8443,node03.xuzhihao.net:8443 \
  --broker-service-url pulsar://node01.xuzhihao.net:6650,node02.xuzhihao.net:6650,node03.xuzhihao.net:6650 \
  --broker-service-url-tls pulsar+ssl://node01.xuzhihao.net::6651,node02.xuzhihao.net:6651,node03.xuzhihao.net:6651

./pulsar-admin clusters get cluster-1       # 获取集群配置
./pulsar-admin clusters update cluster-1 --url http://node01.xuzhihao.net:8080 --broker-url pulsar://node01.xuzhihao.net:6650       # 修改集群配置
./pulsar-admin clusters delete cluster-1    # 删除集群
```

### 3.2 Tenants租户

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-admin tenants list                 # 获取租户列表
./pulsar-admin tenants create my-tenant     # 创建租户
./pulsar-admin tenants create my-tenant -r role1
./pulsar-admin tenants create my-tenant --admin-roles role1,role2,role3     # 可以使用-r 或者 --admin-roles标志分配管理角色
./pulsar-admin tenants get my-tenant        # 获取租户配置
./pulsar-admin tenants update my-tenant -r "dev"     # 修改租户
./pulsar-admin tenants delete my-tenant     # 删除租户，如果库下已经有名称空间, 是无法删除的，需要先删除名称空间
```

### 3.3 Brokers

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-admin brokers list use             # 获取所有可用的brokers
./pulsar-admin brokers leader-broker        # 获取leader broker的信息
```

### 3.4 NameSpace命名空间

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-admin namespaces create my-tenant/test-namespace       # 创建命名空间
./pulsar-admin namespaces list my-tenant                        # 获取所有的名称空间列表
./pulsar-admin namespaces delete my-tenant/test-namespace       # 删除命名空间
./pulsar-admin namespaces policies my-tenant/test-namespace     # 获取名称空间相关的配置策略

# 集群配置
./pulsar-admin namespaces set-clusters my-tenant/test-namespace --clusters cl2      # 设置复制集群，cl2为目标集群
./pulsar-admin namespaces get-clusters my-tenant/test-namespace                     # 获取给定命名空间复制集群的列表

# 配置backlog quota策略
./pulsar-admin namespaces set-backlog-quota --limit 10G --limitTime 36000 --policy producer_request_hold my-tenant/test-namespace
./pulsar-admin namespaces get-backlog-quotas my-tenant/test-namespace       # 获取backlog quota策略
./pulsar-admin namespaces remove-backlog-quota my-tenant/test-namespace     # 移除backlog quota策略
# 参数说明：
# producer_request_hold:broker暂停运行，并且不再持久化生产请求负载。
# producer_exception:broker抛出异常，并与客户端断开连接。
# consumer_backlog_eviction:broker丢弃积压消息。

# 设置持久化策略
./pulsar-admin namespaces set-persistence --bookkeeper-ack-quorum 2 --bookkeeper-ensemble 3 --bookkeeper-write-quorum 2 --ml-mark-delete-max-rate 0 my-tenant/test-namespace    
pulsar-admin namespaces get-persistence my-tenant/test-namespace     # 获取持久化策略
# 参数说明：
# bookkeeper-ack-quorum:每个entry在等待的acks(有保证的副本)数量，默认值：0
# bookkeeper-ensemble:单个topic使用的bookie数量，默认值：0
# bookkeeper-write-quorum：每个entry要写入的次数，默认值：0
# ml-mark-delete-max-rate:标记删除操作的限制速率（0.0表示无限制），默认值：0.0

# 设置消息存活策略
./pulsar-admin namespaces set-message-ttl --messageTTL 100 my-tenant/test-namespace     # 设置消息存活时间
./pulsar-admin namespaces get-message-ttl my-tenant/test-namespace                      # 获取消息的存活时间
./pulsar-admin namespaces remove-message-ttl my-tenant/test-namespace                   # 删除消息的存活时间

# 消息速率设置
./pulsar-admin namespaces set-dispatch-rate my-tenant/test-namespace --msg-dispatch-rate 1000 --byte-dispatch-rate 1048576 --dispatch-rate-period 1     # 设置Topic的消息发送的速率
./pulsar-admin namespaces get-dispatch-rate my-tenant/test-namespace    # 获取消息发送的速率
# 参数说明: 
# msg-dispatch-rate 每秒钟发送的消息数量
# byte-dispatch-rate 每秒钟发送的总字节数
# dispatch-rate-period 设置发送的速率，比如1表示每秒钟

./pulsar-admin namespaces set-subscription-dispatch-rate my-tenant/test-namespace --msg-dispatch-rate 1000 --byte-dispatch-rate 1048576 --dispatch-rate-period 1    # 设置Topic的消息接收的速率
./pulsar-admin namespaces get-subscription-dispatch-rate my-tenant/test-namespace   # 获取消息接收的速率

./pulsar-admin namespaces set-replicator-dispatch-rate my-tenant/test-namespace --msg-dispatch-rate 1000 --byte-dispatch-rate 1048576 --dispatch-rate-period 1      # 设置Topic的消息复制集群的速率
./pulsar-admin namespaces get-replicator-dispatch-rate my-tenant/test-namespace     # 获取Topic的消息复制集群的速率
```

### 3.5 Permissions授权

```bash
cd /opt/apache-pulsar-2.10.1/bin
./pulsar-admin namespaces grant-permission my-tenant/test-namespace --actions produce,consume --role admin10    # 授予权限
./pulsar-admin namespaces permissions my-tenant/test-namespace                          # 获取权限
./pulsar-admin namespaces revoke-permission my-tenant/test-namespace --role admin10     # 撤销权限
```

### 3.6 Topic主题

```bash
cd /opt/apache-pulsar-2.10.1/bin
persistent://tenant/namespace/topic         # 持久化topic地址的命名格式
non-persistent://tenant/namespace/topic     # 非持久化topic地址的命名格式
./pulsar-admin topics list my-tenant/test-namespace     # 列出当前某个名称空间下的所有Topic

./pulsar-admin topics create persistent://my-tenant/test-namespace/my-topic   # 创建一个没有分区的topic
./pulsar-admin topics delete persistent://my-tenant/test-namespace/my-topic   # 删除没有分区的topic

./pulsar-admin topics create-partitioned-topic persistent://my-tenant/test-namespace/my-topic2 --partitions 4       # 创建一个有分区的topic
# 注意：不管有没有分区，topic创建后，如果没有任何操作，60秒后pulsar会认为此topic是不活动的，会自动进行删除，以避免产生垃圾数据。
# brokerdeleteinactivetopicsenabenabled:默认值为true，表示是否自动删除topic
# brokerdeleteinactivetopicsfrequencyseconds:默认值为60s，表示检测未活动的时间。
./pulsar-admin topics update-partitioned-topic persistent://my-tenant/test-namespace/my-topic2 --partitions 8       # 更新topic分区的数量
./pulsar-admin topics delete-partitioned-topic persistent://my-tenant/test-namespace/my-topic2                      # 删除有分区的topic
```

### 3.7 Functions

#### 3.7.1 修改配置

```bash
cd /opt/apache-pulsar-2.10.1/conf
vim broker.conf
# 编辑内容
functionsWorkerEnabled=true   # 三台节点都需要调整
```

重启服务

```bash
bin/pulsar-daemon stop broker
bin/pulsar-daemon start broker
```

#### 3.7.2 创建function

```bash
cd /opt/apache-pulsar-2.10.1
bin/pulsar-admin functions create \
--jar examples/api-examples.jar \
--classname org.apache.pulsar.functions.api.examples.ExclamationFunction \
--inputs persistent://public/default/exclamation-input \
--output persistent://public/default/exclamation-output \
--tenant public \
--namespace default \
--name hello-fun
```

```bash

bin/pulsar-admin functions update --tenant public --namespace default --name hello-fun --output persistent://public/default/update-output-topic   # 修改函数
bin/pulsar-admin functions start --tenant public --namespace default --name hello-fun       # 启动函数
bin/pulsar-admin functions stop --tenant public --namespace default --name hello-fun        # 停止函数
bin/pulsar-admin functions restart --tenant public --namespace default --name hello-fun     # 重启函数
bin/pulsar-admin functions list --tenant public --namespace default                         # 列出所有函数
bin/pulsar-admin functions delete --tenant public --namespace default --name hello-fun      # 删除函数
bin/pulsar-admin functions stats --tenant public --namespace default --name hello-fun       # 获取函数统计信息
```

#### 3.7.3 检查触发函数

```bash
bin/pulsar-admin functions trigger --name hello-fun --trigger-value "hello world"
```

#### 3.7.4 启动消费者

```bash
cd /opt/apache-pulsar-2.10.1
bin/pulsar-client consume persistent://public/default/exclamation-output -s 'test'
```


#### 3.7.5 发送测试消息

```bash
cd /opt/apache-pulsar-2.10.1
bin/pulsar-client produce persistent://public/default/exclamation-input --messages '我是个大盗贼'
```

#### 3.7.6 自定义function

代码地址：https://github.com/xzh-net/java-demo/tree/main/pulsar

1. 创建function

```bash
cd /export/server/pulsar_2.8.1/
bin/pulsar-admin functions create \
--jar examples/pulsar-1.0-SNAPSHOT.jar \
--classname net.xzh.pulsar.functions.FormatDateFunction \
--inputs persistent://public/default/test_input \
--output persistent://public/default/test_output \
--tenant public \
--namespace default \
--name FormatDateTest
```

2. 触发测试

```bash
bin/pulsar-admin functions trigger --name FormatDateTest --trigger-value "2022/09/13 23/12/30"
```

3. 启动消费者

```bash
cd /opt/apache-pulsar-2.10.1
bin/pulsar-client consume persistent://public/default/test_output -s 'test'
```


4. 发送测试消息

```bash
cd /opt/apache-pulsar-2.10.1
bin/pulsar-client produce persistent://public/default/test_input --messages '2022/09/13 23/12/30'
```


### 3.8 Package

### 3.9 Transactions