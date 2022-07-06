# Redis 6.2.6

## 1. 安装

### 1.1 单机

#### 1.1.1 下载

```bash
wget https://download.redis.io/releases/redis-6.2.6.tar.gz
```

#### 1.1.2 编译安装

```bash
yum install -y gcc tcl
tar -zxvf redis-6.2.6.tar.gz -C /home
mv /home/redis-6.2.6 /usr/local/redis
cd /usr/local/redis
make PREFIX=/usr/local/redis install
```

#### 1.1.3 修改配置

```bash
cd /usr/local/redis
mkdir -p /usr/local/redis/conf /usr/local/redis/data
cat redis.conf | grep -v "#" | grep -v "^$" > conf/redis-6379.conf
echo '' > conf/redis-6379.conf 
vi conf/redis-6379.conf

port 6379
daemonize yes
protected-mode no
logfile "6379.log"
dir /usr/local/redis/data
```

#### 1.1.4 启动服务

```bash
cd /usr/local/redis/bin
./redis-server ../conf/redis-6379.conf	# 指定配置启动
./redis-server --port 6380				# 默认配置指定端口启动
```

#### 1.1.5 客户端测试

```bash
/usr/local/redis/bin/redis-cli -p 6380
/usr/local/redis/bin/redis-benchmark -n 10000  -q
./redis-benchmark -h 127.0.0.1 -p 6379 -t set,lpush -n 10000 -q
```

### 1.2 主从

#### 1.2.1 主配置6379

```bash
vi /usr/local/redis/conf/redis-6379.conf
```

```bash
port 6379
daemonize yes
protected-mode no
logfile "6379.log"
dir /usr/local/redis/data/6379
rdbcompression yes
rdbchecksum yes
save 3600 1
save 300 100
save 60 10000
appendonly no
appendfilename "appendonly-6379.aof"
```

```bash
sed 's/6379/6380/g' redis-6379.conf > redis-6380.conf
```

#### 1.2.2 从配置6380

```bash
vi /usr/local/redis/conf/redis-6380.conf
```

```bash
port 6380
daemonize yes
protected-mode no
logfile "6380.log"
dir /usr/local/redis/data/6380
rdbcompression yes
rdbchecksum yes
save 3600 1
save 300 100
save 60 10000
appendonly no
appendfilename "appendonly-6380.aof"
replicaof 192.168.3.200 6379
```

#### 1.2.3 从配置6381

```bash
vi /usr/local/redis/conf/redis-6381.conf
```

```bash
port 6381
daemonize yes
protected-mode no
logfile "6381.log"
dir /usr/local/redis/data/6381
rdbcompression yes
rdbchecksum yes
save 3600 1
save 300 100
save 60 10000
appendonly no
appendfilename "appendonly-6381.aof"
replicaof 192.168.3.200 6379
```


#### 1.2.4 启动服务

```bash
cd /usr/local/redis/bin
./redis-server ../conf/redis-6379.conf
./redis-server ../conf/redis-6380.conf
./redis-server ../conf/redis-6381.conf
```


### 1.3 哨兵

#### 1.3.1 主配置26379

```bash
vi /usr/local/redis/conf/sentinel-26379.conf
```

```bash
port 26379
daemonize yes
protected-mode no
logfile "26379.log"
dir /usr/local/redis/data/26379
sentinel monitor mymaster 192.168.3.200 6379 2
sentinel auth-pass mymaster 123456
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

#### 1.3.2 从配置26380

```bash
vi /usr/local/redis/conf/sentinel-26380.conf
```

```bash
port 26380
daemonize yes
protected-mode no
logfile "26380.log"
dir /usr/local/redis/data/26380
sentinel monitor mymaster 192.168.3.200 6379 2
sentinel auth-pass mymaster 123456
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

#### 1.3.3 从配置26381

```bash
vi /usr/local/redis/conf/sentinel-26381.conf
```

```bash
port 26381
daemonize yes
protected-mode no
logfile "26381.log"
dir /usr/local/redis/data/26381
sentinel monitor mymaster 192.168.3.200 6379 2
sentinel auth-pass mymaster 123456
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

#### 1.3.4 启动服务

```bash
cd /usr/local/redis/bin
./redis-sentinel ../conf/sentinel-26379.conf
./redis-sentinel ../conf/sentinel-26380.conf
./redis-sentinel ../conf/sentinel-26381.conf
```

### 1.4 集群

#### 1.4.1 配置文件

```bash
cd /usr/local/redis/data
mkdir 6379 6380 6381 26379 26380 26381
```

```bash
vi /usr/local/redis/conf/redis-6379.conf
```

```bash
port 6379
daemonize yes
protected-mode no
logfile "6379.log"
dir /usr/local/redis/data/6379
cluster-enabled yes
cluster-config-file /usr/local/redis/data/6379/nodes.conf
cluster-node-timeout 5000
databases 1
```

#### 1.4.2 批量启动

```bash
cd /usr/local/redis/data
# 拷贝文件
printf '%s\n' 6379 6380 6381 26379 26380 26381 | xargs -I{} -t cp /usr/local/redis/conf/redis-6379.conf {}/redis-{}.conf
# 修改文件
printf '%s\n' 6379 6380 6381 26379 26380 26381 | xargs -I{} -t sed -i 's/6379/{}/g' {}/redis-{}.conf
# 启动
printf '%s\n' 6379 6380 6381 26379 26380 26381 | xargs -I{} -t /usr/local/redis/bin/redis-server {}/redis-{}.conf
# 关闭
ps aux|grep redis
ps -ef | grep redis | awk '{print $2}' | xargs kill
```

#### 1.4.3 初始化

```bash
cd /usr/local/redis/bin
./redis-cli --cluster create --cluster-replicas 1 192.168.3.201:6379 192.168.3.201:6380 192.168.3.201:6381 192.168.3.201:26379 192.168.3.201:26380 192.168.3.201:26381

./redis-cli -c -p 6379 cluster nodes
```


## 2. 命令

```bash
# String可实现：`缓存，限流，计数器，分布式锁，session共享`
SETEX key seconds value          # 将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)。
MSETNX key value [key value ...] # 同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。
# 非原子
SETNX key value                 # 只有在 key 不存在时设置 key 的值。
GETRANGE key start end          # 返回 key 中字符串值的子字符
MSET key value [key value ...]  # 同时设置一个或多个 key-value 对。
SET key value [EX] [PX] [NX|XX] # 设置指定 key 的值
Get key                         # 获取指定 key 的值。
GETBIT key offset               # 对 key 所储存的字符串值，获取指定偏移量上的位(bit)。
SETBIT key offset value         # 对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit)。
DECR key                        # 将 key 中储存的数字值减一。
DECRBY key decrement            # key 所储存的值减去给定的减量值（decrement） 。
Strlen key                      # 返回 key 所储存的字符串值的长度。
INCR key                        # 将 key 中储存的数字值增一。
INCRBY key increment            # 将 key 所储存的值加上给定的增量值（increment） 。
INCRBYFLOAT key increment       # 将 key 所储存的值加上给定的浮点增量值（increment） 。
SETRANGE key offset value       # 用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始。
PSETEX key milliseconds value   # 这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。
APPEND key value                # 如果 key 已经存在并且是一个字符串， APPEND 命令将 value 追加到 key 原来的值的末尾。
GETSET key value                # 将给定 key 的值设为 value ，并返回 key 的旧值(old value)。
MGET key [key ...]              # 获取所有(一个或多个)给定 key 的值。

# Hash可实现：`用户信息存储，访问量等组合查询`
HMSET key field value [field value ...] # 同时将多个 field-value (域-值)对设置到哈希表 key 中。
HMGET key field [field ...]             # 获取所有给定字段的值
HSET key field value                    # 将哈希表 key 中的字段 field 的值设为 value 。
HGETALL key                             # 获取在哈希表中指定 key 的所有字段和值
HGET key field                          # 获取存储在哈希表中指定字段的值/td>
HEXISTS key field                       # 查看哈希表 key 中，指定的字段是否存在。
HINCRBY key field increment             # 为哈希表 key 中的指定字段的整数值加上增量 increment 。
HINCRBYFLOAT key field increment        # 为哈希表 key 中的指定字段的浮点数值加上增量 increment 。
HLEN key                        # 获取哈希表中字段的数量
HDEL key field [field ...]      # 删除一个或多个哈希表字段
HVALS key                       # 获取哈希表中所有值
HKEYS key                       # 获取所有哈希表中的字段
HSETNX key field value          # 只有在字段 field 不存在时，设置哈希表字段的值。

# List可实现：`取最新n个，简单队列`
RPOPLPUSH source destination            # 移除列表的最后一个元素，并将该元素添加到另一个列表并返回
# 非原子
LINDEX key index                        # 通过索引获取列表中的元素
RPUSH key value [value ...]             # 在列表中添加一个或多个值插入到列表 key 的表尾(最右边)
LRANGE key start stop                   # 获取列表指定范围内的元素
BLPOP key [key ...] timeout             # 移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
BRPOP key [key ...] timeout             # 移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
BRPOPLPUSH source destination timeout   # 阻塞 从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
LREM key count value                # 移除列表元素
LLEN key                            # 获取列表长度
LTRIM key start stop                # 对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。
LPOP key                            # 移出并获取列表的第一个元素
LPUSHX key value                    # 将一个或多个值插入到已存在的列表头部
RPOP key                            # 移除并获取列表最后一个元素
LSET key index value                # 通过索引设置列表元素的值
LPUSH key value [value ...]         # 将一个或多个值插入到列表头部
RPUSHX key value                    # 为已存在的列表添加值
LINSERT key BEFORE|AFTER pivot value # 在列表的元素前或者后插入元素

#  Set可实现：`共同好友，交集差集，踩，赞，标签`
SUNION key [key ...]            # 返回所有给定集合的并集
SCARD key                       # 获取集合的成员数
SRANDMEMBER key [count]         # 返回集合中一个或多个随机数
SMEMBERS key                    # 返回集合中的所有成员
SINTER key [key ...]            # 返回给定所有集合的交集
SREM key member [member ...]    # 移除集合中一个或多个成员
SMOVE source destination member # 将 member 元素从 source 集合移动到 destination 集合
SADD key member [member ...]    # 向集合添加一个或多个成员
SISMEMBER key member            # 判断 member 元素是否是集合 key 的成员
SDIFF key [key ...]             # 返回给定所有集合的差集
SPOP key                        # 移除并返回集合中的一个随机元素
SSCAN key cursor [MATCH pattern] [COUNT count] # 迭代集合中的元素
SDIFFSTORE destination key [key ...]            # 返回给定所有集合的差集并存储在 destination 中
SINTERSTORE destination key [key ...]           # 返回给定所有集合的交集并存储在 destination 中
SUNIONSTORE destination key [key ...]           # 所有给定集合的并集存储在 destination 集合中

# Sorted Set 可实现：`排行榜`
ZCARD key                           # 获取有序集合的成员数
ZSCORE key member                   # 返回有序集中，成员的分数值
ZRANK key member                    # 返回有序集合中指定成员的索引
ZCOUNT key min max                  # 计算在有序集合中指定区间分数的成员数
ZREVRANK key member                 # 返回有序集合中指定成员的排名，有序集成员按分数值递减(从大到小)排序
ZREM key member [member ...]        # 移除有序集合中的一个或多个成员
ZREMRANGEBYRANK key start stop      # 移除有序集合中给定的排名区间的所有成员
ZREMRANGEBYSCORE key min max        # 移除有序集合中给定的分数区间的所有成员
ZINCRBY key increment member        # 有序集合中对指定成员的分数加上增量 increment
ZRANGE key start stop [WITHSCORES]  # 通过索引区间返回有序集合成指定区间内的成员
ZUNIONSTORE destination numkeys key [key ...]                   # 计算给定的一个或多个有序集的并集，并存储在新的 key 中
ZINTERSTORE destination numkeys key [key ...]                   # 计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]     # 通过分数返回有序集合指定区间内的成员
ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]  # 返回有序集中指定分数区间内的成员，分数从高到低排序
ZREVRANGE key start stop [WITHSCORES]                           # 返回有序集中指定区间内的成员，通过索引，分数从高到底
ZSCAN key cursor [MATCH pattern] [COUNT count]                  # 迭代有序集合中的元素（包括元素成员和元素分值）
ZADD key score member [[score member] [score member] ...]       # 向有序集合添加一个或多个成员，或者更新已存在成员的分数

# 键命令
DEL key [key ...]       # 该命令用于在 key 存在是删除 key。
DUMP key                # 序列化给定 key ，并返回被序列化的值。
EXISTS key              # 检查给定 key 是否存在。
EXPIRE key seconds      # seconds 为给定 key 设置过期时间。
KEYS pattern            # 查找所有符合给定模式( pattern)的 key 。
MOVE key db             # 将当前数据库的 key 移动到给定的数据库 db 当中。
PERSIST key             # 移除 key 的过期时间，key 将持久保持。
RENAME key newkey       # 修改 key 的名称
RENAMENX key newkey     # 仅当 newkey 不存在时，将 key 改名为 newkey 。
RANDOMKEY               # 从当前数据库中随机返回一个 key 。
Type key                # 返回 key 所储存的值的类型。
EXPIREAT key timestamp  # 设置 key 过期时间的时间戳(unix timestamp)
TTL key                 # 以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)。
Pttl key                # 以毫秒为单位返回 key 的剩余的过期时间。
PEXPIREAT key milliseconds-timestamp # 设置 key 的过期时间亿以毫秒计。

# 连接命令
Ping # 查看服务是否运行
Quit # 关闭当前连接
ECHO message  # 打印字符串
SELECT index  # 切换到指定的数据库
AUTH password # 验证密码是否正确

# 服务器命令
redis-server --version
redis-server /opt/redis/redis.conf
redis-cli -h host -p port -a password
CLIENT PAUSE timeout        # 在指定时间内终止运行来自客户端的命令
DEBUG OBJECT key            # 获取 key 的调试信息
FLUSHDB                     # 删除当前数据库的所有key
SAVE                        # 同步以RDB文件的形式保存到硬盘
BGSAVE                      # 在后台异步保存当前数据库的数据到磁盘
BGREWRITEAOF                # 异步执行一个 AOF（AppendOnly File） 文件重写操作
Flushall                    # 删除所有数据库的所有key
Dbsize                      # 返回当前数据库的 key 的数量
Cluster Slots               # 获取集群节点的映射数组
Config Set                  # 修改 配置参数，无需重启
LASTSAVE                    # 返回最近一次 成功将数据保存到磁盘上的时间，以 UNIX 时间戳格式表示
CONFIG GET parameter        # 获取指定配置参数的值
COMMAND                     # 获取 命令详情数组
SLAVEOF host port           # 将当前服务器转变为指定服务器的从属服务器(slave server)
Debug Segfault              # 执行一个非法的内存访问从而让 Redis 崩溃，仅在开发时用于 BUG 调试
SHUTDOWN [NOSAVE] [SAVE]    # 异步保存数据到硬盘，并关闭服务器
SYNC                        # 用于复制功能(replication)的内部命令
CLIENT KILL ip:port         # 关闭客户端连接
ROLE                        # 返回主从实例所属的角色
MONITOR                     # 实时打印出 服务器接收到的命令，调试用
COMMAND GETKEYS             # 获取给定命令的所有键
CLIENT GETNAME              # 获取连接的名称
CONFIG RESETSTAT            # 重置 INFO 命令中的某些统计数据
COMMAND COUNT               # 获取 命令总数
TIME                        # 返回当前服务器时间
INFO                        # 获取 服务器的各种信息和统计数值
CONFIG REWRITE parameter    # 对启动服务器时所指定的 redis.conf 配置文件进行改写
CLIENT LIST                 # 获取连接到服务器的客户端连接列表
CLIENT SETNAME connection-name  # 设置当前连接的名称
SLOWLOG subcommand [argument]   # 管理的慢日志
COMMAND INFO command-name [command-name ...]  # 获取指定 命令描述的数组

# 脚本命令
Script kill             # 杀死当前正在运行的 Lua 脚本。
SCRIPT LOAD script      # 将脚本 script 添加到脚本缓存中，但并不立即执行这个脚本。
SCRIPT FLUSH            # 从脚本缓存中移除所有脚本。
EVAL script numkeys key [key ...] arg [arg ...]     # 执行 Lua 脚本。
EVALSHA sha1 numkeys key [key ...] arg [arg ...]    # 执行 Lua 脚本。
SCRIPT EXISTS script [script ...]                   # 查看指定的脚本是否已经被保存在缓存当中。

# 事务命令
Exec                # 执行所有事务块内的命令。
Unwatch             # 取消 WATCH 命令对所有 key 的监视。
WATCH key [key ...] # 监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。
Discard             # 取消事务，放弃执行事务块内的所有命令。
Multi               # 标记一个事务块的开始。

# HyperLogLog命令
PFMERGE destkey sourcekey [sourcekey ...] # 将多个 HyperLogLog 合并为一个 HyperLogLog
PFADD key element [element ...] # 添加指定元素到 HyperLogLog 中。
PFCOUNT key [key ...] # 返回给定 HyperLogLog 的基数估算值。

# 发布订阅命令
UNSUBSCRIBE [channel [channel ...]] # 指退订给定的频道。
SUBSCRIBE channel [channel ...] # 订阅给定的一个或多个频道的信息。
PUBSUB <subcommand> [argument [argument ...]] # 查看订阅与发布系统状态。
PUNSUBSCRIBE [pattern [pattern ...]] # 退订所有给定模式的频道。
PUBLISH channel message # 将信息发送到指定的频道。
PSUBSCRIBE pattern [pattern ...] # 订阅一个或多个符合给定模式的频道。

# 地理位置(geo)命令
GEOHASH # 返回一个或多个位置元素的 Geohash 表示
GEOPOS # 从key里返回所有给定位置元素的位置（经度和纬度）
GEODIST # 返回两个给定位置之间的距离
GEORADIUS # 以给定的经纬度为中心， 找出某一半径内的元素
GEOADD # 将指定的地理空间位置（纬度、经度、名称）添加到指定的key中
GEORADIUSBYMEMBER # 找出位于指定范围内的元素，中心点是由给定的位置元素决定
```

## 3. 企业级解决方案

### 3.1 设计规范

1. key的规范要点
  - 以业务名为key前缀，用冒号隔开，以防止key冲突覆盖。如，`live:rank:1`
  - 确保key的语义清晰的情况下，key的长度尽量小于30个字符。拒绝bigkey
  - key禁止包含特殊字符，如空格、换行、单双引号以及其他转义字符。

2. value的规范要点
  - string类型控制在10KB以内，hash、list、set、zset元素个数不要超过5000
  - 要选择适合的数据类型
  - 使用expire设置过期时间(条件允许可以打散过期时间，防止集中过期)，不过期的数据重点关注idletime

3. 命令使用
  - [推荐]禁止线上使用keys、flushall、flushdb等，通过redis的rename机制禁掉命令，或者使用scan渐进式处理
  - [推荐]使用pipeline批量操作提高效率
  - [推荐]O(N)命令关注N的数量,hgetall、lrange、smembers、zrange、sinter等并非不能使用，但是需要明确N的值。有遍历的需求可以使用hscan、sscan、zscan代替
  - [建议]Redis的事务功能较弱(不支持回滚)，而且集群版本(自研和官方)要求一次事务操作的key必须在一个slot上(可以使用hashtag功能解决)不建议过多使用
  - [建议]集群版本Lua上有特殊要求:所有key，必须在1个slot上
  - [建议]必要情况下使用monitor命令时，要注意不要长时间使用

4. 运维

- Redis Cluster只支持db0，切换会损耗新能，迁移成本高
- 开启 lazy-free机制，减少对主线程的阻塞
- 设置maxmemory + 淘汰策略

为了防止内存积压膨胀。避免直接挂掉，需要根据实际业务，选好maxmemory-policy(最大内存淘汰策略)，设置好过期时间。一共有8种内存淘汰策略：

	1. volatile-lru：当内存不足以容纳新写入数据时，从设置了过期时间的key中使用LRU（最近最少使用）算法进行淘汰
	2. allkeys-lru：当内存不足以容纳新写入数据时，从所有key中使用LRU（最近最少使用）算法进行淘汰
	3. volatile-lfu：4.0版本新增，当内存不足以容纳新写入数据时，在过期的key中，使用LFU算法进行删除key
	4. allkeys-lfu：4.0版本新增，当内存不足以容纳新写入数据时，从所有key中使用LFU算法进行淘汰
	5. volatile-random：当内存不足以容纳新写入数据时，从设置了过期时间的key中，随机淘汰数据
	6. allkeys-random：当内存不足以容纳新写入数据时，从所有key中随机淘汰数据
	7. volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的key中，根据过期时间进行淘汰，越早过期的优先被淘汰
	8. oeviction：默认策略，当内存不足以容纳新写入数据时，新写入操作会报错。

### 3.2 缓存穿透(安全问题)

指查询一个一定不存在的数据，由于缓存是不命中时需要从数据库查询，查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到数据库去查询，进而给数据库带来压力

产生场景：
1. `业务不合理的设计`，比如大多数用户都没开守护，但是你的每个请求都去缓存，查询某个userid查询有没有守护。
2. `业务/运维/开发失误的操作`，比如缓存和数据库的数据都被误删除了。
3. `黑客非法请求攻击`，比如黑客故意捏造大量非法请求，以读取不存在的业务数据

解决方案：
1. 如果是非法请求，我们在API入口，对参数进行校验，过滤非法值。
2. 如果查询数据库为空，我们可以给缓存设置个空值，或者默认值。但是如有有写请求进来的话，需要更新缓存哈，以保证缓存一致性，同时，最后给缓存设置适当的过期时间。（业务上比较常用，简单有效）
3. 使用布隆过滤器快速判断数据是否存在。即一个查询请求过来时，先通过布隆过滤器判断值是否存在，存在才继续往下查

> 布隆过滤器原理：它由初始值为0的位图数组和N个哈希函数组成。一个对一个key进行N个hash算法获取N个值，在比特数组中将这N个值散列后设定为1，然后查的时候如果特定的这几个位置都为1，那么布隆过滤器判断该key存在

`无法确定你是否真的存在，但是可以确定真的不存在。`

```lua
BF.RESERVE	创建一个大小为capacity，错误率为error_rate的空的Bloom	BF.RESERVE {key} {error_rate} {capacity} [EXPANSION expansion] [NONSCALING]
BF.ADD	向key指定的Bloom中添加一个元素item	BF.ADD {key} {item}
BF.MADD	向key指定的Bloom中添加多个元素	BF.MADD {key} {item} [item…]
BF.INSERT	向key指定的Bloom中添加多个元素，添加时可以指定大小和错误率，且可以控制在Bloom不存在的时候是否自动创建	BF.INSERT {key} [CAPACITY {cap}] [ERROR {error}] [EXPANSION expansion] [NOCREATE] [NONSCALING] ITEMS {item…}
BF.EXISTS	检查一个元素是否可能存在于key指定的Bloom中	BF.EXISTS {key} {item}
BF.MEXISTS	同时检查多个元素是否可能存在于key指定的Bloom中	BF.MEXISTS {key} {item} [item…]
BF.SCANDUMP	对Bloom进行增量持久化操作	BF.SCANDUMP {key} {iter}
BF.LOADCHUNK	加载SCANDUMP持久化的Bloom数据	BF.LOADCHUNK {key} {iter} {data}
BF.INFO	查询key指定的Bloom的信息	BF.INFO {key}
BF.DEBUG	查看BloomFilter的内部详细信息（如每层的元素个数、错误率等）	BF.DEBUG {key}
```

### 3.3 缓存雪崩

指缓存中数据大批量到过期时间，而查询数据量巨大，请求都直接访问数据库，引起数据库压力过大甚至down机

解决方案：
1. 均匀设置过期时间解决，即让过期时间相对离散一点。如采用一个较大固定值+一个较小的随机值

### 3.4 缓存击穿

指热点key在某个时间点过期的时候，而恰好在这个时间点对这个Key有大量的并发请求过来，从而大量的请求打到db

解决方案：
1. 使用互斥锁方案。缓存失效时，不是立即去加载db数据，而是先使用某些带成功返回的原子操作命令，如(Redis的setnx）去操作，成功的时候，再去加载db数据库数据和设置缓存。否则就去重试获取缓存。
2. 永不过期，是指没有设置过期时间，但是热点数据快要过期时，异步线程去更新和设置过期时间。


### 3.5 缓存热key

某一热点key的请求到服务器主机时，由于请求量特别大，可能会导致主机资源不足，甚至宕机，从而影响正常的服务

产生场景：
1. 用户消费的数据远大于生产的数据，如秒杀、热点新闻等读多写少的场景
2. 请求分片集中，超过单Redi服务器的性能，比如固定名称key，Hash落入同一台服务器产生数据倾斜，瞬间访问量极大，超过机器瓶颈

如何识别：
- 预判和统计

解决方案：
1. Redis集群扩容
2. 对热key进行hash散列，比如将一个key备份为key1,key2……keyN，同样的数据N个备份，N个备份分布到不同分片，访问时可随机访问N个备份中的一个，进一步分担读流量；
3. 使用二级缓存，即JVM本地缓存,减少Redis的读请求

### 3.6 数据倾斜

即热点 key，指的是在一段时间内，该 key 的访问量远远高于其他的 redis key， 导致大部分的访问流量在经过 proxy 分片之后，都集中访问到某一个 redis 实例上

解决方案：
1. 利用分片算法的特性，对key进行打散处理

给hot key加上前缀或者后缀，把一个hotkey 的数量变成 redis 实例个数N的倍数M，从而由访问一个 redis key 变成访问 N * M 个redis key。N*M 个 redis key 经过分片分布到不同的实例上，将访问量均摊到所有实例

```lua
//redis 实例数
const M = 16
 
//redis 实例数倍数（按需设计，2^n倍，n一般为1到4的整数）
const N = 2
 
func main() {
//获取 redis 实例 
    c, err := redis.Dial("tcp", "127.0.0.1:6379")
    if err != nil {
        fmt.Println("Connect to redis error", err)
        return
    }
    defer c.Close()
 
    hotKey := "hotKey:abc"
    //随机数
    randNum := GenerateRangeNum(1, N*M)
    //得到对 hot key 进行打散的 key
    tmpHotKey := hotKey + "_" + strconv.Itoa(randNum)
 
    //hot key 过期时间
    expireTime := 50
 
    //过期时间平缓化的一个时间随机值
    randExpireTime := GenerateRangeNum(0, 5)
 
    data, err := redis.String(c.Do("GET", tmpHotKey))
    if err != nil {
        data, err = redis.String(c.Do("GET", hotKey))
        if err != nil {
            data = GetDataFromDb()
            c.Do("SET", "hotKey", data, expireTime)
            c.Do("SET", tmpHotKey, data, expireTime + randExpireTime)
        } else {
            c.Do("SET", tmpHotKey, data, expireTime + randExpireTime)
        }
    }
}
```

### 3.7 分布不均问题、Hash Tags

1. 问题原理

- 对于客户端请求的key，根据公式HASH_SLOT=CRC16(key) mod 16384，计算出映射到哪个分片上，然后Redis会去相应的节点进行操作
- keySlot算法中，如果key包含{}，就不对整个key做hash，而是使用第一个{}内部的字符串作为hash key，这样就可以保证拥有同样{}内部字符串的key就会拥有相同slot

单实例上的MSET是一个原子性(atomic)操作，所有给定 key 都会在同一时间内被设置，某些给定 key 被更新而另一些给定 key 没有改变的情况，不可能发生

而集群上虽然也支持同时设置多个key，但不再是原子性操作。会存在某些给定 key 被更新而另外一些给定 key 没有改变的情况。其原因是需要设置的多个key可能分配到不同的机器上。

SINTERSTORE,SUNIONSTORE,ZINTERSTORE,ZUNIONSTORE

这四个命令属于同一类型。它们的共同之处是都需要对一组key进行运算或操作，但要求这些key都被分配到相同机器上。

这就是分片技术的矛盾之处：

即要求key尽可能地分散到不同机器，又要求某些相关联的key分配到相同机器

2. Hash Tags

HashTag机制可以影响key被分配到的slot，从而可以使用那些被限制在slot中操作

HashTag即是用{}包裹key的一个子串，如{user:}1, {user:}2，在设置了HashTag的情况下，集群会根据HashTag决定key分配到的slot， 两个key拥有相同的HashTag:{user:}, 它们会被分配到同一个slot，允许我们使用MGET命令

HashTag不支持嵌套，可能会使过多的key分配到同一个slot中，造成数据倾斜影响系统的吞吐量，务必谨慎使用


### 3.8 bigkey删除

- Hash删除: hscan + hdel

```java
public void delBigHash(String host, int port, String password, String bigHashKey) {
		Jedis jedis = new Jedis(host, port);
		if (password != null && !"".equals(password)) {
			jedis.auth(password);
		}
		ScanParams scanParams = new ScanParams().count(100);
		String cursor = "0";
		do {
			ScanResult<Entry<String, String>> scanResult = jedis.hscan(bigHashKey, cursor, scanParams);
			List<Entry<String, String>> entryList = scanResult.getResult();
			if (entryList != null && !entryList.isEmpty()) {
				for (Entry<String, String> entry : entryList) {
					jedis.hdel(bigHashKey, entry.getKey());
				}
			}
			cursor = scanResult.getStringCursor();
		} while (!"0".equals(cursor));

		// 删除bigkey
		jedis.del(bigHashKey);
	}
```

- List删除: ltrim

```java
public void delBigList(String host, int port, String password, String bigListKey) {
		Jedis jedis = new Jedis(host, port);
		if (password != null && !"".equals(password)) {
			jedis.auth(password);
		}
		long llen = jedis.llen(bigListKey);
		int counter = 0;
		int left = 100;
		while (counter < llen) {
			// 每次从左侧截掉100个
			jedis.ltrim(bigListKey, left, llen);
			counter += left;
		}
		// 最终删除key
		jedis.del(bigListKey);
	}
```

- Set删除: sscan + srem

```java
public void delBigSet(String host, int port, String password, String bigSetKey) {
		Jedis jedis = new Jedis(host, port);
		if (password != null && !"".equals(password)) {
			jedis.auth(password);
		}
		ScanParams scanParams = new ScanParams().count(100);
		String cursor = "0";
		do {
			ScanResult<String> scanResult = jedis.sscan(bigSetKey, cursor, scanParams);
			List<String> memberList = scanResult.getResult();
			if (memberList != null && !memberList.isEmpty()) {
				for (String member : memberList) {
					jedis.srem(bigSetKey, member);
				}
			}
			cursor = scanResult.getStringCursor();
		} while (!"0".equals(cursor));

		// 删除bigkey
		jedis.del(bigSetKey);
	}
```

- SortedSet删除: zscan + zrem

```java
public void delBigZset(String host, int port, String password, String bigZsetKey) {
		Jedis jedis = new Jedis(host, port);
		if (password != null && !"".equals(password)) {
			jedis.auth(password);
		}
		ScanParams scanParams = new ScanParams().count(100);
		String cursor = "0";
		do {
			ScanResult<Tuple> scanResult = jedis.zscan(bigZsetKey, cursor, scanParams);
			List<Tuple> tupleList = scanResult.getResult();
			if (tupleList != null && !tupleList.isEmpty()) {
				for (Tuple tuple : tupleList) {
					jedis.zrem(bigZsetKey, tuple.getElement());
				}
			}
			cursor = scanResult.getStringCursor();
		} while (!"0".equals(cursor));

		// 删除bigkey
		jedis.del(bigZsetKey);
	}
```

## 4. shell



### 4.1 批量删除

```bash
redis-cli -h 127.0.0.1 -p 6379 -a "password"

redis-cli -a "password" -n 0 -p 6379 keys "*:login" | xargs -i redis-cli -a "password" -n 0 -p 6379 del {}	# 缺点：每次建立一个连接
redis-cli -a "password" -n 0 -p 6379 --scan --pattern "user:admin:info:*" | xargs -i redis-cli -a "password" -n 0 -p 6379 DEL {}  # 模式匹配快速删除，速度快
redis-cli -c -h 172.17.17.193 -p 26379 -a "password" keys "GetMerchantByUserId*" | xargs -r -t -n1 redis-cli -c -h 172.17.17.193 -p 26379 -a "password" del
```

### 4.2 批量添加key 

vi add.sh

```bash
#!/bin/bash

redis_list=("172.17.17.193:26379:redis26379")

pkey_list=("ValuationRuleSummary" "ValuationRuleDetail" "MerchantValuation" "QueryValuationRules" "GetMerchantByUserId")

for info in ${redis_list[@]}
do
    echo "开始执行:$info"  
    ip=`echo $info | cut -d \: -f 1`
    port=`echo $info | cut -d \: -f 2`
	pwd=`echo $info | cut -d \: -f 3`

	for pkey in ${pkey_list[@]}
	do
		for i in `seq 0 1000`
		do
			echo $pkey"_"$i 	$i
			redis-cli -c -h $ip -p $port -a $pwd SET $pkey"_"$i $i
		done
	done
done
echo "完成"
```

### 4.3 批量删除key 

vi del.sh

```bash
#!/bin/bash

redis_list=("10.1.37.113:7000" "10.1.37.113:7001" "10.1.37.113:7002" "10.1.37.113:7003" "10.1.37.113:7004" "10.1.37.113:7005")

pkey_list=("ValuationRuleSummary*" "ValuationRuleDetail*" "MerchantValuation*" "QueryValuationRules*" "GetMerchantByUserId*")

for info in ${redis_list[@]}
    do
        echo "开始执行:$info"  
        ip=`echo $info | cut -d \: -f 1`
        port=`echo $info | cut -d \: -f 2`

		for pkey in ${pkey_list[@]}
		do
			redis-cli -c -h $ip -p $port -a PASSWORD KEYS $pkey | xargs -n 1 -t -i redis-cli -c -h $ip -p $port -a PASSWORD DEL {}
		done
    done
echo "完成"
```

### 4.4 通过keys文件批量操作

编辑待删除的key，vi key.txt

```lua
key1
key2
key3
```

```bash
redis_list=("127.0.0.1:6379" "127.0.0.1:6380")

for info in ${redis_list[@]}
    do  
        echo "开始执行:$info"  
        ip=`echo $info | cut -d \: -f 1`
        port=`echo $info | cut -d \: -f 2`
        cat key.txt |xargs -t -n1 redis-cli -h $ip -p $port -c del  
    done 
echo "完成"
```
