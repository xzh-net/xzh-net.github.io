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


## 2. 插件管理

目前 EMQX 发行包提供的插件包括：

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

## 3. 认证

### 3.1 认证简介

#### 3.1.1 认证方式

EMQX支持的认证方式：

内置数据源
- Mnesia (用户名/Client ID）认证

外部数据源
- LDAP 认证
- MySQL 认证
- PostgreSQL 认证
- Redis 认证
- MongoDB 认证

其他
- HTTP 认证
- JWT 认证

#### 3.1.2 匿名登录

EMQX默认配置中启用了匿名认证，任何客户端都能接入EMQX。没有启用认证插件或认证插件没有显式允许/拒绝（ignore）连接请求时，EMQX将根据匿名认证启用情况决定是否允许客户端连接

配置匿名认证开关：
```conf
# etc/emqx.conf
## Value: true | false
allow_anonymous = false
```

#### 3.1.3 密码加盐规则与哈希方法

EMQX 多数认证插件中可以启用哈希方法，数据源中仅保存密码密文，保证数据安全。

启用哈希方法时，用户可以为每个客户端都指定一个 salt（盐）并配置加盐规则，数据库中存储的密码是按照加盐规则与哈希方法处理后的密文。

以 MySQL 认证为例，加盐规则与哈希方法配置：

```sh
# etc/plugins/emqx_auth_mysql.conf

## 不加盐，仅做哈希处理
auth.mysql.password_hash = sha256

## salt 前缀：使用 sha256 加密 salt + 密码 拼接的字符串
auth.mysql.password_hash = salt,sha256

## salt 后缀：使用 sha256 加密 密码 + salt 拼接的字符串
auth.mysql.password_hash = sha256,salt

## pbkdf2 with macfun iterations dklen
## macfun: md4, md5, ripemd160, sha, sha224, sha256, sha384, sha512
## auth.mysql.password_hash = pbkdf2,sha256,1000,20
```

#### 3.1.4 认证流程

1. 根据配置的认证SQL结合客户端传入的信息，查询出密码（密文）和salt（盐）等认证数据，没有查询结果时，认证将终止并返回ignore结果
2. 根据配置的加盐规则与哈希方法计算得到密文，没有启用哈希方法则跳过此步
3. 将数据库中存储的密文与当前客户端计算的到的密文进行比对，比对成功则认证通过，否则认证失败（更改哈希方法会造成已有认证数据失效）
4. 当同时启用多个认证方式时，EMQX将按照插件开启先后顺序进行链式认证，如果认证成功，终止认证链并允许客户端接入。一旦认证失败，跳过当前认证方式进入下一个，直到最后一个认证方式仍未通过，根据匿名认证配置判定。
5. 生产环境只启用一个认证插件可以提高客户端身份认证效率


#### 3.1.5 TLS认证

TLS的默认端口是8883，可通过`etc/emqx.conf`修改

```conf
listener.ssl.external = 8883
listener.ssl.external.keyfile = etc/certs/key.pem
listener.ssl.external.certfile = etc/certs/cert.pem
listener.ssl.external.cacertfile = etc/certs/cacert.pem
```

注意，默认的`etc/certs`目录下面的 `key.pem`、`cert.pem` 和 `cacert.pem` 是EMQX生成的自签名证书，所以在使用支持TLS的客户端测试的时候，需要将上面的CA证书`etc/certs/cacert.pem` 配置到客户端。

服务端支持的cipher列表需要显式指定，默认的列表与Mozilla的服务端cipher列表一致：

```conf
listener.ssl.external.ciphers = ECDHE-ECDSA-AES256-GCM-SHA384,ECDHE-RSA-AES256-GCM-SHA384,ECDHE-ECDSA-AES256-SHA384,ECDHE-RSA-AES256-SHA384,ECDHE-ECDSA-DES-CBC3-SHA,ECDH-ECDSA-AES256-GCM-SHA384,ECDH-RSA-AES256-GCM-SHA384,ECDH-ECDSA-AES256-SHA384,ECDH-RSA-AES256-SHA384,DHE-DSS-AES256-GCM-SHA384,DHE-DSS-AES256-SHA256,AES256-GCM-SHA384,AES256-SHA256,ECDHE-ECDSA-AES128-GCM-SHA256,ECDHE-RSA-AES128-GCM-SHA256,ECDHE-ECDSA-AES128-SHA256,ECDHE-RSA-AES128-SHA256,ECDH-ECDSA-AES128-GCM-SHA256,ECDH-RSA-AES128-GCM-SHA256,ECDH-ECDSA-AES128-SHA256,ECDH-RSA-AES128-SHA256,DHE-DSS-AES128-GCM-SHA256,DHE-DSS-AES128-SHA256,AES128-GCM-SHA256,AES128-SHA256,ECDHE-ECDSA-AES256-SHA,ECDHE-RSA-AES256-SHA,DHE-DSS-AES256-SHA,ECDH-ECDSA-AES256-SHA,ECDH-RSA-AES256-SHA,AES256-SHA,ECDHE-ECDSA-AES128-SHA,ECDHE-RSA-AES128-SHA,DHE-DSS-AES128-SHA,ECDH-ECDSA-AES128-SHA,ECDH-RSA-AES128-SHA,AES128-SHA
```

### 3.2 Mnesia 认证

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

### 3.3 HTTP 认证

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

### 3.4 JWT 认证

### 3.5 LDAP 认证

### 3.6 MySQL 认证

### 3.7 PostgreSQL 认证

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


### 3.8 Redis 认证

### 3.9 MongoDB 认证

## 4. 发布订阅ACL

### 4.1 发布订阅ACL简介
