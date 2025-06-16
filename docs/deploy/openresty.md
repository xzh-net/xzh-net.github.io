# OpenResty 1.25

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

#### 1.4.1 hello World

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


#### 1.4.2 获取http请求信息

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


#### 1.4.3 操作redis

将redis.lua文件复制到`/usr/local/openresty/lualib/resty/`目录下

下载地址：https://openresty.org/en/lua-resty-redis-library.html


```lua
local redis = require "resty.redis"
local red = redis:new()

red:set_timeout(1000) -- 1 sec

local ok, err = red:connect("172.17.17.192", 16386)
if not ok then
    ngx.say("failed to connect: ", err)
    return
end

-- 这里是 auth 的调用过程
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

local headers = ngx.req.get_headers()
-- 获取token
local res, err = red:get(headers['token'])
if not res then
    ngx.say("请登录")
    return
end

if res == ngx.null then
    ngx.say("请登录.")
    return
end

ngx.say(res)

-- 连接池大小是100个，并且设置最大的空闲时间是 10 秒
local ok, err = red:set_keepalive(10000, 100)
if not ok then
    ngx.say("failed to set keepalive: ", err)
    return
end
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
