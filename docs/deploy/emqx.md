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

#### 1.1.2 启动服务

```bash
/opt/emqx/bin
./emqx start
```

控制台地址：http://192.168.2.201:18083/
默认账号密码admin/public

#### 1.1.3 客户端测试

下载客户端： https://www.emqx.com/zh/downloads/MQTTX/1.9.10/MQTTX-Setup-1.9.10-x64.exe

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

配置哈希方法后，新增的预设认证数据与通过HTTP API添加的认证数据将以哈希密文存储在EMQX内置数据库中（使用VSCode安装REST Client插件）

```conf
@hostname = 192.168.2.201
@port=18083
@contentType=application/json
@userName=admin
@password=Aa000000

#############查看已有用户认证数据##############
GET http://{{hostname}}:{{port}}/api/v4/auth_username HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

########添加用户认证数据##############
POST http://{{hostname}}:{{port}}/api/v4/auth_username HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

{
    "username": "xuzhihao",
    "password": "123456"
}

###########更改指定用户名的密码#############
PUT http://{{hostname}}:{{port}}/api/v4/auth_username/user HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

{
    "password": "user"
}

###########查看指定用户名信息#############
GET http://{{hostname}}:{{port}}/api/v4/auth_username/user HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

###########删除指定的用户信息#############
DELETE http://{{hostname}}:{{port}}/api/v4/auth_username/user HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}
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

### 2.5 HTTP API

### 2.6 数据集成

### 2.7 消息桥接

### 2.8 规则引擎


