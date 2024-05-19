# Emqx 4.4.19

EMQX基于Erlang/OTP平台开发的MQTT消息服务器，是开源社区中最流行的MQTT消息服务器。

- 网站地址：https://www.emqx.io/zh/community

## 1. 安装

### 1.1 单机

#### 1.1.1 上传解压

```bash
cd /opt/software
wget https://www.emqx.com/zh/downloads/broker/4.4.19/emqx-4.4.19-otp24.3.4.2-1-el7-amd64.zip
unzip emqx-4.4.19-otp24.3.4.2-1-el7-amd64.zip
mv emqx /opt
```

#### 1.1.2 关闭交换分区

Linux 交换分区可能会导致Erlang虚拟机出现不确定的内存延迟，严重影响系统的稳定性。 建议永久关闭交换分区。
- 要立即关闭交换分区，执行命令 sudo swapoff -a
- 要永久关闭交换分区，在`/etc/fstab`文件中注释掉swap行，然后重新启动主机

#### 1.1.3 系统参数设置

系统全局允许分配的最大文件句柄数

```bash
vi /etc/sysctl.conf
# 设置参数
fs.file-max = 2097152
fs.nr_open = 2097152
```

```bash
sysctl -p
```

设置服务最大文件句柄数

```bash
vi /etc/systemd/system.conf
# 设置参数
DefaultLimitNOFILE=1048576
```

```bash
systemctl daemon-reexec
```

持久化设置允许用户/进程打开文件句柄数

```bash
vi /etc/security/limits.conf

*      soft   nofile      1048576
*      hard   nofile      1048576
```


#### 1.1.4 TCP网络参数

并发连接backlog设置
```bash
sysctl -w net.core.somaxconn=32768
sysctl -w net.ipv4.tcp_max_syn_backlog=16384
sysctl -w net.core.netdev_max_backlog=16384
```

端口范围设置
```bash
sysctl -w net.ipv4.ip_local_port_range='1024 65535'
```

TCP Socket读写Buffer设置
```bash
sysctl -w net.core.rmem_default=262144
sysctl -w net.core.wmem_default=262144
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216 
sysctl -w net.core.optmem_max=16777216
sysctl -w net.ipv4.tcp_rmem='1024 4096 16777216'
sysctl -w net.ipv4.tcp_wmem='1024 4096 16777216'
```

TCP 连接追踪设置
```bash
sysctl -w net.nf_conntrack_max=1000000
sysctl -w net.netfilter.nf_conntrack_max=1000000
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
```

TIME-WAIT Socket最大数量、回收与重用设置
```bash
sysctl -w net.ipv4.tcp_max_tw_buckets=1048576
```

FIN-WAIT-2 Socket 超时设置
```bash
sysctl -w net.ipv4.tcp_fin_timeout=15
```

#### 1.1.5 Erlang虚拟机参数

优化设置 Erlang 虚拟机启动参数，配置文件`etc/emqx.conf`:

```conf
## 设置 Erlang 系统同时存在的最大端口数
node.max_ports = 2097152
```

#### 1.1.6 EMQX消息服务器参数

设置 TCP 监听器的 Acceptor 池大小，最大允许连接数

```conf
## TCP Listener
listeners.tcp.$name.acceptors = 64
listeners.tcp.$name.max_connections = 1024000
```

#### 1.1.7 启动服务

```bash
/opt/emqx/bin
./emqx start
```

控制台地址：http://192.168.2.201:18083/
默认账号密码admin/public

#### 1.1.8 客户端测试

- 下载客户端： https://www.emqx.com/zh/downloads/MQTTX/1.9.10/MQTTX-Setup-1.9.10-x64.exe
- 并发连接测试工具: https://github.com/emqx/emqtt-bench

### 1.2 集群

## 2. 高级

### 2.1 插件

目前EMQX发行包提供的插件包括：

| 插件                      | 配置文件       | 说明                                                         |
| ------------------------- | ---------- | ------------------------------------------------------------ |
| emqx_dashboard | 	etc/plugins/emqx_dashbord.conf  | 	Web 控制台插件 (默认加载)|
| emqx_management | 	etc/plugins/emqx_management.conf  | 	HTTP API and CLI 管理插件|
| emqx_auth_mnesia | 	etc/plugins/emqx_auth_mnesia.conf  | 	Mnesia 认证 / 访问控制|
| emqx_auth_jwt | 	etc/plugins/emqx_auth_jwt.conf  | 	JWT 认证 / 访问控制|
| emqx_auth_ldap | 	etc/plugins/emqx_auth_ldap.conf  | 	LDAP 认证 / 访问控制|
| emqx_auth_http | 	etc/plugins/emqx_auth_http.conf  | 	HTTP API 与 CLI 管理插件|
| emqx_auth_mongo | 	etc/plugins/emqx_auth_mongo.conf  | 	MongoDB 认证 / 访问控制|
| emqx_auth_mysql | 	etc/plugins/emqx_auth_mysql.conf  | 	MySQL 认证 / 访问控制|
| emqx_auth_pgsql | 	etc/plugins/emqx_auth_pgsql.conf  | 	PostgreSQL 认证 / 访问控制|
| emqx_auth_redis | 	etc/plugins/emqx_auth_redis.conf  | 	Redis 认证 / 访问控制|
| emqx_psk_file | 	etc/plugins/emqx_psk_file.conf  | 	PSK 支持|
| emqx_web_hook | 	etc/plugins/emqx_web_hook.conf  | 	Web Hook 插件|
| emqx_lua_hook | 	etc/plugins/emqx_lua_hook.conf  | 	Lua Hook 插件|
| emqx_retainer | 	etc/plugins/emqx_retainer.conf  | 	Retain 消息存储模块|
| emqx_rule_engine | 	etc/plugins/emqx_rule_engine.conf  | 	规则引擎|
| emqx_bridge_mqtt | 	etc/plugins/emqx_bridge_mqtt.conf  | 	MQTT 消息桥接插件|
| emqx_coap | 	etc/plugins/emqx_coap.conf  | 	CoAP 协议支持|
| emqx_lwm2m | 	etc/plugins/emqx_lwm2m.conf  | 	LwM2M 协议支持|
| emqx_sn | 	etc/plugins/emqx_sn.conf  | 	MQTT-SN 协议支持|
| emqx_stomp | 	etc/plugins/emqx_stomp.conf  | 	Stomp 协议支持|
| emqx_recon | 	etc/plugins/emqx_recon.conf  | 	Recon 性能调试|
| emqx_plugin_template | 	etc/plugins/emqx_plugin_template.conf  | 	代码热加载插件|

目前启动插件有以下四种方式：
1. 默认加载

    在`data/loaded_plugins`中添加需要启动的插件名称（`有先后顺序，在认证中体现`）
2. 命令行启停插件

    在 EMQX 运行过程中，可通过CLI的方式查看、和启停某插件
3. 使用Dashboard启停插件

    若开启了Dashboard 的插件，可以直接通过访问 http://localhost:18083/plugins 中的插件管理页面启停插件。
4. 调用管理API启停插件

    EMQX的HTTP API服务默认监听8081端口，可通过`etc/plugins/emqx_management.conf` 配置文件修改监听端口，或启用HTTPS监听。EMQX 4.0以后的所有API调用均以api/v4开头

### 2.2 认证

EMQX默认配置中启用了匿名认证，任何客户端都能接入EMQX。没有启用认证插件或认证插件没有显式允许/拒绝（ignore）连接请求时，EMQX将根据匿名认证启用情况决定是否允许客户端连接

配置匿名认证开关：
```conf
# etc/emqx.conf
## Value: true | false
allow_anonymous = false
```

#### 2.2.1 Mnesia 认证

Mnesia认证使用EMQX内置Mnesia数据库存储客户端Client ID/Username与密码，支持通过HTTP API管理认证数据。

Mnesia认证默认使用sha256进行密码哈希加密，可在 `etc/plugins/emqx_auth_mnesia.conf` 中更改：

```conf
# etc/plugins/emqx_auth_mnesia.conf

## Value: plain | md5 | sha | sha256 
auth.mnesia.password_hash = sha256
```

预设认证数据，预设认证数据格式兼容 emqx_auth_clientid 与 emqx_auth_username 插件的配置格式

```conf
# etc/plugins/emqx_auth_mnesia.conf

## clientid 认证数据
auth.client.1.clientid = admin
auth.client.1.password = public

## username 认证数据
auth.user.2.username = admin
auth.user.2.password = public
```

#### 2.2.2 HTTP 认证

使用外部自建HTTP应用认证数据源，根据API返回的数据判定认证结果，能够实现复杂的认证鉴权逻辑。在请求中传递明文密码，加盐规则与哈希方法取决于HTTP应用。

通过返回的HTTP响应状态码(HTTP statusCode) 来处理认证请求
- 认证失败：API 返回非 200 状态码
- 认证成功：API 返回 200 状态码
- 忽略认证：API 返回 200 状态码且消息体 ignore

```conf
# etc/plugins/emqx_auth_http.conf

## 请求地址
auth.http.auth_req = http://127.0.0.1:80/mqtt/auth

## HTTP 请求方法
## Value: post | get | put
auth.http.auth_req.method = post

## 认证请求的 HTTP 请求头部，默认情况下配置 Content-Type 头部。
## Content-Type 头部目前支持以下值：application/x-www-form-urlencoded，application/json
auth.http.auth_req.headers.content-type = application/x-www-form-urlencoded

## 请求参数
auth.http.auth_req.params = clientid=%c,username=%u,password=%P,ip=%a
```

#### 2.2.3 JWT 认证

#### 2.2.4 LDAP 认证

#### 2.2.5 MySQL 认证

#### 2.2.6 PostgreSQL 认证

要启用PostgreSQL认证，需要在`etc/plugins/emqx_auth_pgsql.conf`中配置以下内容：

```conf
# etc/plugins/emqx_auth_pgsql.conf

## 服务器地址
auth.pgsql.server = 127.0.0.1:5432

## 连接池大小
auth.pgsql.pool = 8
auth.pgsql.username = root
auth.pgsql.password = public
auth.pgsql.database = mqtt
auth.pgsql.encoding = utf8

## TLS 配置
## auth.pgsql.ssl = false
## auth.pgsql.ssl_opts.keyfile =
## auth.pgsql.ssl_opts.certfile =
```

默认表结构

```sql
CREATE TABLE mqtt_user (
  id SERIAL PRIMARY KEY,
  username CHARACTER VARYING(100),
  password CHARACTER VARYING(100),
  salt CHARACTER VARYING(40),
  is_superuser BOOLEAN,
  UNIQUE (username)
)
```

默认配置下示例数据如下：

```sql
INSERT INTO mqtt_user (username, password, salt, is_superuser)
VALUES
	('emqx', 'efa1f375d76194fa51a3556a97e641e61685f914d446979da50a551a4333ffd7', NULL, false);
```

> 启用PostgreSQL认证后，你可以通过用户名： emqx，密码：public 连接。

加盐规则与认证SQL

```conf
# etc/plugins/emqx_auth_pgsql.conf
auth.pgsql.password_hash = sha256
auth.pgsql.auth_query = select password from mqtt_user where username = '%u' limit 1
```


#### 2.2.7 Redis 认证

#### 2.2.8 MongoDB 认证

### 2.3 发布订阅ACL

默认配置中ACL是开放授权的，即授权结果为忽略（ignore）时允许客户端通过授权。

通过`etc/emqx.conf`中的AC 配置可以更改该属性：

```conf
# etc/emqx.conf

## ACL 未匹配时默认授权
## Value: allow | deny
acl_nomatch = allow
```

配置默认ACL文件，使用文件定义默认ACL规则：

```conf
# etc/emqx.conf

acl_file = etc/acl.conf
```

配置ACL授权结果为禁止的响应动作，为disconnect时将断开设备：

```conf
# etc/emqx.conf

## Value: ignore | disconnect
acl_deny_action = ignore
```

> 在 MQTT v3.1和v3.1.1协议中，发布操作被拒绝后服务器无任何报文错误返回，这是协议设计的一个缺陷。但在 MQTT v5.0协议上已经支持应答一个相应的错误报文。

#### 2.3.1 内置 ACL

#### 2.3.2 Mnesia ACL

#### 2.3.3 HTTP ACL

#### 2.3.4 JWT ACL

#### 2.3.5 MySQL ACL

#### 2.3.6 PostgreSQL ACL

#### 2.3.7 Redis 认证

#### 2.3.8 MongoDB 认证

### 2.4 WebHook 

WebHook是由`emqx_web_hook`插件提供的将EMQ X中的钩子事件通知到某个Web服务的功能。

Webhook的配置文件位于`etc/plugins/emqx_web_hook.conf`

```conf
web.hook.url = http://127.0.0.1:80/mqtt/webhook
web.hook.headers.content-type = application/json
web.hook.body.encoding_of_payload_field = plain
web.hook.pool_size = 32
web.hook.enable_pipelining = true
web.hook.rule.client.connect.1       = {"action": "on_client_connect"}
web.hook.rule.client.connack.1       = {"action": "on_client_connack"}
web.hook.rule.client.connected.1     = {"action": "on_client_connected"}
web.hook.rule.client.disconnected.1  = {"action": "on_client_disconnected"}
web.hook.rule.client.subscribe.1     = {"action": "on_client_subscribe"}
web.hook.rule.client.unsubscribe.1   = {"action": "on_client_unsubscribe"}
web.hook.rule.session.subscribed.1   = {"action": "on_session_subscribed"}
web.hook.rule.session.unsubscribed.1 = {"action": "on_session_unsubscribed"}
web.hook.rule.session.terminated.1   = {"action": "on_session_terminated"}
web.hook.rule.message.publish.1      = {"action": "on_message_publish"}
web.hook.rule.message.delivered.1    = {"action": "on_message_delivered"}
web.hook.rule.message.acked.1        = {"action": "on_message_acked"}
```

### 2.5 HTTP API

EMQ X提供了HTTP API以实现与外部系统的集成，例如查询客户端信息、发布消息和创建规则等。

EMQ X的HTTP API服务默认监听8081 端口，可通过`etc/plugins/emqx_management.conf`配置文件修改监听端口，或启用HTTPS监听。EMQX 4.0.0以后的所有API调用均以api/v4 开头。

```conf
management.listener.http = 8081
management.listener.http.acceptors = 2
management.listener.http.max_clients = 512
management.listener.http.backlog = 512
management.listener.http.send_timeout = 15s
management.listener.http.send_timeout_close = on
management.listener.http.inet6 = false
management.listener.http.ipv6_v6only = false
```


调用实例：使用VSCode安装`REST Client`插件

```conf
@hostname = 192.168.2.201
@port=18083
@contentType=application/json
@userName=admin
@password=Aa000000

#############查看已有用户认证数据#############
GET http://{{hostname}}:{{port}}/api/v4/auth_username HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

#############添加用户认证数据#############
POST http://{{hostname}}:{{port}}/api/v4/auth_username HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

{
    "username": "xuzhihao",
    "password": "123456"
}

#############更改指定用户名的密码#############
PUT http://{{hostname}}:{{port}}/api/v4/auth_username/user HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

{
    "password": "user"
}

#############查看指定用户名信息#############
GET http://{{hostname}}:{{port}}/api/v4/auth_username/user HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

#############删除指定的用户信息#############
DELETE http://{{hostname}}:{{port}}/api/v4/auth_username/user HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}
```

### 2.6 保留消息

EMQ X的保留消息功能是由emqx_retainer插件实现，该插件默认开启，通过修改emqx_retainer插件的配置，可以调整 EMQ X 储存保留消息的位置，限制接收保留消息数量和Payload最大长度，以及调整保留消息的
过期时间。

emqx_retainer插件默认开启，插件的配置路径为`etc/plugins/emqx_retainer.conf`。

```conf
retainer.storage_type = ram
retainer.max_retained_messages = 0
retainer.max_payload_size = 1MB
retainer.expiry_interval = 0
```

> EMQ X Enterprise中可将保留消息存储到多种外部数据库。

### 2.7 共享订阅

共享订阅是在多个订阅者之间实现负载均衡的订阅方式，注意：共享订阅的主题格式是针对订阅端来指定的。

- 以`$share/<group-name>`为前缀的共享订阅是带群组的共享订阅
- 以`$queue/`为前缀的共享订阅是不带群组的共享订阅。它是$share订阅的一种特例，相当与所有订阅者都在一个订阅组里面

EMQ X的共享订阅支持均衡策略与派发Ack配置：

```conf
# etc/emqx.conf

# 均衡策略
## Dispatch strategy for shared subscription
##
## Value: Enum
## - random
## - round_robin
## - sticky
## - hash
broker.shared_subscription_strategy = random
# 共享分发时是否需要 ACK，适用于 QoS1 QoS2 消息，启用时，当通过shared_subscription_strategy选中的一个订阅者离线时，应该允许将消息发送到组中的另一个订阅者
broker.shared_dispatch_ack_enabled = false
```

### 2.8 延迟发布

EMQ X的延迟发布功能可以实现按照用户配置的时间间隔延迟发布 PUBLISH 报文的功能。当客户端使用特殊主题前缀`$delayed/{DelayInteval}`发布消息到EMQ X时，将触发延迟发布功能。延迟发布的功能是针对消息发布者而言的，订阅方只需要按照正常的主题订阅即可。

延迟发布功能由`emqx_mod_delayed`内置模块提供，此功能默认开启，支持动态启停。

例如:
- $delayed/15/x/y: 15 秒后将 MQTT 消息发布到主题 x/y。
- $delayed/60/a/b: 1 分钟后将 MQTT 消息发布到 a/b。
- $delayed/3600/$SYS/topic: 1 小时后将 MQTT 消息发布到 $SYS/topic

### 2.9 代理订阅

EMQ X 的代理订阅功能使得客户端在连接建立时，不需要发送额外的 SUBSCRIBE 报文，便能自动建立用户预设的订阅关系。

代理订阅功能由`emqx_mod_subscription`内置模块提供，此功能默认关闭，支持在EMQ X Broker运行期间动态启停。

```conf
# etc/emqx.conf

module.subscription.1.topic = client/%c
module.subscription.1.qos = 1

module.subscription.2.topic = user/%u
module.subscription.2.qos = 2

module.subscription.3.topic = testtopic/#
module.subscription.3.qos = 2
```

### 2.10 主题重写

EMQ X的主题重写功能支持根据用户配置的规则在客户端订阅主题、发布消息、取消订阅的时候将A主题重写为B主题。

主题重写功能默认关闭，开启此功能需要启用`emqx_mod_rewrite`模块

```conf
# etc/emqx.conf

module.rewrite.sub.rule.1 = y/+/z/# ^y/(.+)/z/(.+)$ y/z/$2
module.rewrite.pub.rule.1 = x/# ^x/y/(.+)$ z/y/x/$1
module.rewrite.pub.rule.2 = x/y/+ ^x/y/(\d+)$ z/y/$1
```

### 2.11 黑名单

EMQ X为用户提供了黑名单功能，用户可以通过相关的HTTP API将指定客户端加入黑名单以拒绝该客户端访问，除了客户端标识符以外，还支持直接封禁用户名甚至IP地址。

在黑名单功能的基础上，EMQ X支持自动封禁那些被检测到短时间内频繁登录的客户端，并且在一段时间内拒绝这些客户端的登录，以避免此类客户端过多占用服务器资源而影响其他客户端的正常使用。

需要注意的是，自动封禁功能只封禁客户端标识符，并不封禁用户名和IP地址，即该机器只要更换客户端标识符就能够继续登录。

此功能默认关闭，用户可以在`emqx.conf`配置文件中将`enable_flapping_detect`配置项设为on以启用此功能。

```conf
zone.external.enable_flapping_detect = off
# 客户端在 1 分钟内离线次数达到 30 次，那么该客户端使用的客户端标识符将被封禁 5 分钟
flapping_detect_policy = 30, 1m, 5m
```

### 2.12 速率限制

EMQ X提供对接入速度、消息速度的限制：当客户端连接请求速度超过指定限制的时候，暂停新连接的建立；当消息接收速度超过指定限制的时候，暂停接收消息。

速率限制是一种backpressure方案，从入口处避免了系统过载，保证了系统的稳定和可预测的吞吐。速率限制可在`etc/emqx.conf`中配置：

```conf
# 单个 emqx 节点上连接建立的速度限制。1000 代表秒最多允许 1000 个客户端接入
listener.tcp.external.max_conn_rate = 1000
# 单个连接上接收 PUBLISH 报文的速率限制。100,10s 代表每个连接上允许收到的最大 PUBLISH 消息速率是每 10 秒 100 个
zone.external.rate_limit.conn_messages_in = 100,10s
# 单个连接上接收 TCP数据包的速率限制。100KB,10s 代表每个连接上允许收到的最大 TCP 报文速率是每 10 秒 100KB
zone.external.rate_limit.conn_bytes_in = 100KB,10s
```

### 2.13 规则引擎


