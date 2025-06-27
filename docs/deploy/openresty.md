# OpenResty 1.25.3.1

- 官方网站：https://openresty.org

## 1. 安装

### 1.1 安装依赖

```bash
yum install -y pcre-devel openssl-devel gcc curl zlib-devel readline-devel
```

### 1.2 下载编译

```bash
cd /opt/software
wget https://openresty.org/download/openresty-1.25.3.1.tar.gz
tar -zxf openresty-1.25.3.1.tar.gz
cd openresty-1.25.3.1
./configure
make && make install
```

### 1.3 设置环境变量

```bash
vi /etc/profile
export PATH=$PATH:/usr/local/openresty/nginx/sbin
source /etc/profile
```

### 1.4 入门案例

#### 1.4.1 Hello World

```bash
cd /usr/local/openresty/nginx/conf
vi nginx.conf
```

```conf
location / {
    default_type 'text/html';
    content_by_lua_block {
        ngx.say("hello World");
    }
}
```

也可以使用`content_by_lua_file`来引入一个lua文件

```conf
location / {
    default_type 'text/html';
    content_by_lua_file /usr/local/openresty/nginx/conf/hello.lua;
}
```

```lua
ngx.say("hello World");
```


#### 1.4.2 读取请求参数

```lua
-- 获取get请求参数
local arg = ngx.req.get_uri_args()
for k,v in pairs(arg) do
    ngx.say("[get] key:", k, " v:", v)
    ngx.say("<br>")
end

-- 获取post请求参数
ngx.req.read_body()
local arg = ngx.req.get_post_args()
for k,v in pairs(arg) do
    ngx.say("[post] key:", k, " v:", v)
    ngx.say("<br>")
end

-- 获取header
local headers = ngx.req.get_headers()
for k,v in pairs(headers) do
    ngx.say("[header] name:", k, " v:", v)
    ngx.say("<br>")
end

-- 获取body信息
ngx.req.read_body()
local data = ngx.req.get_body_data()
ngx.say(data)
```


```bash
curl -H "Content-Type: application/json" -X POST -d '{"id": "001", "name":"张三", "phone":"13099999999"}'  http://192.168.2.201
```


#### 1.4.3 操作Redis

下载客户端：https://openresty.org/en/lua-resty-redis-library.html

解压后将redis.lua文件复制到`/usr/local/openresty/lualib/resty/`目录下。业务场景：读取cookie中指定名称字段获取凭证，再调用redis查询用户信息


```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  logs/access.log  main;
    sendfile        on; 
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  localhost;
        charset utf-8;
		location / {
			add_header Content-Type 'text/html; charset=utf-8';
            return 200 "你好，当前时间：$time_local";
        }
		
		# 登录页面 - 设置Cookie
		location /login {
			default_type 'text/html';
            content_by_lua_block {
                local token = "zhangsan" 
                ngx.header["Set-Cookie"] = "auth_token=" .. token .. "; Path=/; HttpOnly"
                ngx.say("登录成功，已设置凭证")
            }
        }
		
		# 获取用户 - 需要验证token
		location /current {
			default_type 'text/html';
			content_by_lua_file conf.d/user.lua;
		}
		
		# 登出 - 清除Cookie
        location /logout {
			default_type 'text/html';
            content_by_lua_block {
                ngx.header["Set-Cookie"] = "auth_token=; Path=/; Expires=Thu, 01 Jan 1970 00:00:00 GMT"
                ngx.say("登出成功，已清除凭证")
            }
        }
    }
}
```

```lua
-- 从Cookie中提取auth_token
local cookie = ngx.var.http_cookie
local auth_token

if cookie then
	-- 使用正则表达式匹配auth_token
	local m, err = ngx.re.match(cookie, "auth_token=([^;]+)", "jo")
	if m then
		auth_token = m[1]
	end
end

-- 如果未找到token，返回401错误
if not auth_token then
	ngx.status = ngx.HTTP_UNAUTHORIZED
	ngx.say("没有凭证，请登录")
	return ngx.exit(401)
end

ngx.say(auth_token)

local redis = require "resty.redis"
local red = redis:new()
red:set_timeout(1000)   -- 设置超时时间

local ok, err = red:connect("172.17.17.192", 16386)
red:select(1)           -- 选择数据库1
if not ok then
    ngx.say("failed to connect: ", err)
    return
end

-- 这里是 auth_token 的验证过程
local count
count, err = red:get_reused_times()
if 0 == count then
    ok, err = red:auth("123456")
    if not ok then
        ngx.say("failed to auth: ", err)
        return
    end
elseif err then
    ngx.say("failed to get reused times: ", err)
    return
end

-- 获取token
local res, err = red:get(auth_token)
if not res then
    ngx.say("系统错误，请重试")
	red:set_keepalive(10000, 100)
    return
end

if res == ngx.null then
    ngx.say("会话过期，请登录")
	red:set_keepalive(10000, 100)
    return
end
ngx.say("验证成功")
```

#### 1.4.4 探测网站状态

下载并安装`lua-resty-http`库

```bash
wget https://github.com/ledgetech/lua-resty-http/archive/refs/tags/v0.17.2.tar.gz
tar -zxvf lua-resty-http-0.17.2.tar.gz
sudo cp -r lua-resty-http-0.17.2/lib/resty /usr/local/openresty/lualib/
```

Nginx配置

```conf
server {
    listen 80;
    listen [::]:80;

    server_name _;
    
    location /check_health {
        default_type 'text/html';
        add_header Content-Type 'text/html; charset=utf-8';
        content_by_lua_file conf/conf.d/check_health.lua;
    }
}
```

lua脚本内容

```lua
local http = require "resty.http"
local url = ngx.var.arg_url  -- 获取url参数

-- 创建 HTTP 客户端
local client = http.new()
client:set_timeout(5000)  -- 设置超时(毫秒)

-- 发送请求
local res, err = client:request_uri(url, {
    method = "GET",
    headers = { ["User-Agent"] = "Mozilla/5.0" },
    ssl_verify = false  -- 跳过 HTTPS 证书验证
})

-- 处理结果
if not res then
    ngx.say(err)
    ngx.exit(500)
end

-- 检查状态码
if res.status == 200 then
    ngx.say(200)
else
    ngx.say(res.status)
end
```

测试验证
```bash
# 测试正常网站
curl "http://172.17.17.160/check_health?url=https://maven.aliyun.com"
# 测试无效域名
curl "http://172.17.17.160/check_health?url=https://invalid-domain-xxxxxxxx.com"
# 测试超时网站，IP不可达
curl "http://172.17.17.160/check_health??url=http://10.255.255.1"
```

> 出现错误信息`no resolver defined to resolve`和`could not be resolved (110: Operation timed out)`，是因为没有配置DNS解析服务器和设置超时时间。

在Nginx配置文件中Http块内添加以下配置

```conf
# 核心 DNS 配置（必须）
resolver 114.114.114.114 valid=30s;
resolver_timeout 5s;
# 共享内存用于 DNS 缓存
lua_shared_dict dns_cache 5m;
```

## 2. 模块

1. `init_by_lua*`

该指令在每次Nginx重新加载配置时执行，可以用来完成一些耗时模块的加载，或者初始化一些全局配置。

```lua
init_by_lua_block{
    redis = require "resty.redis"
    mysql = require "resty.mysql"
    cjson = require "cjson"
}
```

2. `init_worker_by_lua*`

该指令用于启动一些定时任务，如心跳检查、定时拉取服务器配置等。

3. `set_by_lua*`

该指令只要用来做变量赋值，这个指令一次只能返回一个值，并将结果赋值给Nginx中指定的变量。

4. `rewrite_by_lua*`

该指令用于执行内部URL重写或者外部重定向，典型的如伪静态化URL重写，本阶段在rewrite处理阶段的最后默认执行。

5. `access_by_lua*`

该指令用于访问控制。例如，如果只允许内网IP访问。

6. `content_by_lua*`

该指令是应用最多的指令，大部分任务是在这个阶段完成的，其他的过程往往为这个阶段准备数据，正式处理基本都在本阶段。

```lua
init_by_lua_block{
	redis = require "resty.redis"
    mysql = require "resty.mysql"
    cjson = require "cjson"
}
location / {
        default_type "text/html";
        content_by_lua_block {
            --获取请求的参数username
            local param = ngx.req.get_uri_args()["username"]
            --建立mysql数据库的连接
            local db = mysql:new()
            local ok,err = db:connect{
                host="192.168.200.111",
                port=3306,
                user="root",
                password="123456",
                database="nginx_db"
            }
            if not ok then
                ngx.say("failed connect to mysql:",err)
                return
            end
            --设置连接超时时间
            db:set_timeout(1000)
            --查询数据
            local sql = "";
            if not param then
                sql="select * from users"
            else
                sql="select * from users where username=".."'"..param.."'"
            end
            local res,err,errcode,sqlstate=db:query(sql)
            if not res then
                ngx.say("failed to query from mysql:",err)
                return
            end
            --连接redis
            local rd = redis:new()
            ok,err = rd:connect("192.168.200.111",6379)
            if not ok then
                ngx.say("failed to connect to redis:",err)
                return
            end
            rd:set_timeout(1000)
            --循环遍历数据
            for i,v in ipairs(res) do
                rd:set("user_"..v.username,cjson.encode(v))
            end
            ngx.say("success")
            rd:close()
            db:close()
        }
}
```

7. `header_filter_by_lua*`

该指令用于设置应答消息的头部信息。

8. `body_filter_by_lua*`

该指令是对响应数据进行过滤，如截断、替换。

9. `log_by_lua*`

该指令用于在log请求处理阶段，用Lua代码处理日志，但并不替换原有log处理。

10. `balancer_by_lua*`

该指令主要的作用是用来实现上游服务器的负载均衡器算法

11. `ssl_certificate_by_*`

该指令作用在Nginx和下游服务开始一个SSL握手操作时将允许本配置项的Lua代码。

## 3. 高级

### 3.1 模板渲染

#### 3.1.1 导入模板引擎

下载地址：http://luarocks.org/modules/bungle/lua-resty-template

将template.lua文件复制到`/usr/local/openresty/lualib/resty/`目录下

#### 3.1.2 配置openresty

```conf
server {
    listen 80;
    server_name localhost;
    charset utf-8;
    set $template_root /usr/local/openresty/nginx/html/templates;
    location / {
        default_type text/html;
        content_by_lua_file /usr/local/openresty/nginx/conf/stock.lua;
    }
}
```

#### 3.1.3 设置模板

```bash
vi /usr/local/openresty/nginx/html/templates/demo.html
```

```html
<!DOCTYPE html>
<html>
<body>
  <h1>{{title}}</h1>
  <h2>{{num}}</h2>
</body>
</html>
```

#### 3.1.4 设置解析类库

将redis_iresty.lua文件复制到`/usr/local/openresty/lualib/resty`目录下，修改连接地址

```lua
-- file name: resty/redis_iresty.lua
local redis_c = require "resty.redis"
local ok, new_tab = pcall(require, "table.new")
if not ok or type(new_tab) ~= "function" then
new_tab = function (narr, nrec) return {} end
end
local _M = new_tab(0, 155)
_M._VERSION = '0.01'
local commands = {
"append", "auth", "bgrewriteaof",
"bgsave", "bitcount", "bitop",
"blpop", "brpop",
"brpoplpush", "client", "config",
"dbsize",
"debug", "decr", "decrby",
"del", "discard", "dump",
"echo",
"eval", "exec", "exists",
"expire", "expireat", "flushall",
"flushdb", "get", "getbit",
"getrange", "getset", "hdel",
"hexists", "hget", "hgetall",
"hincrby", "hincrbyfloat", "hkeys",
"hlen",
"hmget", "hmset", "hscan",
"hset",
"hsetnx", "hvals", "incr",
"incrby", "incrbyfloat", "info",
"keys",
"lastsave", "lindex", "linsert",
"llen", "lpop", "lpush",
"lpushx", "lrange", "lrem",
"lset", "ltrim", "mget",
"migrate",
"monitor", "move", "mset",
"msetnx", "multi", "object",
"persist", "pexpire", "pexpireat",
"ping", "psetex", "psubscribe",
"pttl",
"publish", --[[ "punsubscribe", ]] "pubsub",
"quit",
"randomkey", "rename", "renamenx",
"restore",
"rpop", "rpoplpush", "rpush",
"rpushx", "sadd", "save",
"scan", "scard", "script",
"sdiff", "sdiffstore",
"select", "set", "setbit",
"setex", "setnx", "setrange",
"shutdown", "sinter", "sinterstore",
"sismember", "slaveof", "slowlog",
"smembers", "smove", "sort",
"spop", "srandmember", "srem",
"sscan",
"strlen", --[[ "subscribe", ]] "sunion",
"sunionstore", "sync", "time",
"ttl",
"type", --[[ "unsubscribe", ]] "unwatch",
"watch", "zadd", "zcard",
"zcount", "zincrby", "zinterstore",
"zrange", "zrangebyscore", "zrank",
"zrem", "zremrangebyrank", "zremrangebyscore",
"zrevrange", "zrevrangebyscore", "zrevrank",
"zscan",
"zscore", "zunionstore", "evalsha"
}
local mt = { __index = _M }
local function is_redis_null( res )
if type(res) == "table" then
for k,v in pairs(res) do
if v ~= ngx.null then
return false
end
end
return true
elseif res == ngx.null then
return true
elseif res == nil then
return true
end
return false
end
-- change connect address as you need
function _M.connect_mod( self, redis )
redis:set_timeout(self.timeout)
return redis:connect("192.168.2.201", 22121)
end
function _M.set_keepalive_mod( redis )
-- put it into the connection pool of size 100, with 60 seconds max idle time
return redis:set_keepalive(60000, 1000)
end
function _M.init_pipeline( self )
self._reqs = {}
end
function _M.commit_pipeline( self )
local reqs = self._reqs
if nil == reqs or 0 == #reqs then
return {}, "no pipeline"
else
self._reqs = nil
end
local redis, err = redis_c:new()
if not redis then
return nil, err
end
local ok, err = self:connect_mod(redis)
if not ok then
return {}, err
end
redis:init_pipeline()
for _, vals in ipairs(reqs) do
local fun = redis[vals[1]]
table.remove(vals , 1)
fun(redis, unpack(vals))
end
local results, err = redis:commit_pipeline()
if not results or err then
return {}, err
end
if is_redis_null(results) then
results = {}
ngx.log(ngx.WARN, "is null")
end
-- table.remove (results , 1)
self.set_keepalive_mod(redis)
for i,value in ipairs(results) do
if is_redis_null(value) then
results[i] = nil
end
end
return results, err
end
function _M.subscribe( self, channel )
local redis, err = redis_c:new()
if not redis then
return nil, err
end
local ok, err = self:connect_mod(redis)
if not ok or err then
return nil, err
end
local res, err = redis:subscribe(channel)
if not res then
return nil, err
end
res, err = redis:read_reply()
if not res then
return nil, err
end
redis:unsubscribe(channel)
self.set_keepalive_mod(redis)
return res, err
end
local function do_command(self, cmd, ... )
if self._reqs then
table.insert(self._reqs, {cmd, ...})
return
end
local redis, err = redis_c:new()
if not redis then
return nil, err
end
local ok, err = self:connect_mod(redis)
if not ok or err then
return nil, err
end
local fun = redis[cmd]
local result, err = fun(redis, ...)
if not result or err then
-- ngx.log(ngx.ERR, "pipeline result:", result, " err:", err)
return nil, err
end
if is_redis_null(result) then
result = nil
end
self.set_keepalive_mod(redis)
return result, err
end
for i = 1, #commands do
local cmd = commands[i]
_M[cmd] =
function (self, ...)
return do_command(self, cmd, ...)
end
end
function _M.new(self, opts)
opts = opts or {}
local timeout = (opts.timeout and opts.timeout * 1000) or 1000
local db_index= opts.db_index or 0
return setmetatable({
timeout = timeout,
db_index = db_index,
_reqs = nil }, mt)
end
return _M
```

#### 3.1.5 配置解析规则

```bash
vi /usr/local/openresty/nginx/conf/stock.lua
```

```lua
local template = require "resty.template"
local redis = require "resty.redis_iresty"
local red = redis:new()
local msg = red:get("num")
if ngx.null==msg then
    return ngx.say("获取库存失败");
end

local context = {  
    title = "电子商城",
    num=msg
}  
local request_uri = ngx.var.request_uri
template.render(string.sub(request_uri,2),context);
```


#### 3.1.6 访问地址

http://192.168.2.201/demo.html
