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


#### 1.4.2 获取URL查询参数


获取单个参数

```lua
location /api {
    content_by_lua_block {
        local param1 = ngx.var.arg_param1  -- 获取?param1=value 中的值
        local param2 = ngx.var.arg_param2
        ngx.say("param1:", param1)
        ngx.say("param2:", param2)
    }
}
```

测试地址：http://127.0.0.1/api?param1=zhangsan&param2=29


遍历所有参数

```lua
location /api {
    content_by_lua_block {
        local args = ngx.req.get_uri_args()  -- 返回 Lua 表
        for key, val in pairs(args) do
            if type(val) == "table" then
                ngx.say(key, ":", table.concat(val, ","))  -- 处理多值参数
            else
                ngx.say(key, ":", val)
            end
        end
    }
}
```

测试地址：http://127.0.0.1/api?name=zhangsan&age=29&address=beijing,shanghai



#### 1.4.3 获取POST请求体参数

表单数据（application/x-www-form-urlencoded）

```lua
location /login {
    content_by_lua_block {
        ngx.req.read_body()  -- 必须先读取请求体
        local post_args = ngx.req.get_post_args()  -- 解析为 Lua 表

        -- 遍历参数
        for key, val in pairs(post_args) do
            ngx.say(key, ":", val)
        end
    }
}
```

测试

```bash
curl -X POST -d "username=zhangsan" -d "password=123456" http://127.0.0.1:8080/submit
```

JSON 数据 (application/json)


```lua
location /json {
    content_by_lua_block {
        ngx.req.read_body()
        local json_str = ngx.req.get_body_data()  -- 获取原始 JSON 字符串
        local cjson = require "cjson"
        local data = cjson.decode(json_str)  -- 解析为 Lua 表

        ngx.say("username:", data.username)
        ngx.say("password:", data.password)
    }
}
```

测试

```bash
curl -X POST -H "Content-Type: application/json" -d '{"username":"zhangsan", "password":123456}'  http://127.0.0.1:8080/json
```


#### 1.4.4 获取请求头

获取指定头

```lua
location /headers {
    content_by_lua_block {
        -- 直接获取指定头信息（下划线代替连字符）
        local content_type = ngx.var.http_content_type
        local auth_token = ngx.var.http_authorization
        
        ngx.say("Content-Type: ", content_type or "nil")
        ngx.say("Authorization: ", auth_token or "nil")
    }
}
```

获取所有头

```lua
location /headers {
    content_by_lua_block {
        local headers = ngx.req.get_headers()  -- 获取所有请求头
        
        -- 遍历所有头信息
        for key, value in pairs(headers) do
            ngx.say(key, ":", value)
        end
        
        -- 获取特定头信息（不区分大小写）
        local user_agent = headers["User-Agent"] or headers["user-agent"]
        ngx.say("\nUser-Agent: ", user_agent)
    }
}
```



#### 1.4.5 获取Cookie

```lua
location /cookie {
    content_by_lua_block {
        local cookie = ngx.var.http_cookie
        local auth_token

        if cookie then
            -- 使用正则表达式匹配auth_token
            local m, err = ngx.re.match(cookie, "auth_token=([^;]+)", "jo")
            if m then
                auth_token = m[1]
            end
        end
        ngx.say("cookie: ", cookie or "nil")
        ngx.say("auth_token: ", auth_token or "nil")
    }
}
```


#### 1.4.6 操作Redis

下载客户端：https://openresty.org/en/lua-resty-redis-library.html

解压后将redis.lua文件复制到`/usr/local/openresty/lualib/resty/`目录下。

业务场景：拦截所有请求地址，读取cookie中指定名称字段获取凭证，再调用redis查询用户信息，未找到则返回登录页面，`拦截使用了集群模式，获取用户信息使用了单机模式`。

1. 配置nginx.conf

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
        listen 80;
        charset utf-8;
        
        # 拦截所有请求地址
        location / {
            access_by_lua_file conf.d/auth.lua;
            proxy_pass http://172.17.17.165:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			add_header Content-Type "text/html; charset=utf-8";
        }

        # 登录页面 - 设置Cookie
        location = /login {
            default_type 'text/html';
            content_by_lua_block {
                local token = "zhangsan" 
                ngx.header["Set-Cookie"] = "auth_token=" .. token .. "; Path=/; HttpOnly"
                ngx.say("登录成功，已设置凭证")
            }
        }

        # 登出 - 清除Cookie
        location = /logout {
            default_type 'text/html';
            content_by_lua_block {
                ngx.header["Set-Cookie"] = "auth_token=; Path=/; Expires=Thu, 01 Jan 1970 00:00:00 GMT"
                ngx.say("登出成功，已清除凭证")
            }
        }
        
        # 获取用户 - 需要验证token
        location = /current {
            default_type 'text/html';
            content_by_lua_file conf.d/current.lua;
        }
    }

    server {
        listen 8080;
        charset utf-8;
        location / {
            add_header Content-Type 'text/html; charset=utf-8';
            return 200 "首页";
        }
        location /aaa {
            add_header Content-Type 'text/html; charset=utf-8';
            return 200 "aaa";
        }
        location /bbb {
            add_header Content-Type 'text/html; charset=utf-8';
            return 200 "bbb";
        }
    }
}
```


2. 创建脚本auth.lua

```lua
-- 从Cookie中提取auth_token
local function extract_auth_token()
    local cookie = ngx.var.http_cookie
    if not cookie then
        return nil
    end
    
    local m = ngx.re.match(cookie, "auth_token=([^;]+)", "jo")  
    return m and m[1] or nil
end

-- 获取Redis配置
local function get_redis_config()
    return {
        -- 哨兵配置
        sentinels = {
            { host = "172.17.17.161", port = 26379 },
            { host = "172.17.17.161", port = 26380 },
            { host = "172.17.17.161", port = 26381 },
        },
        -- 主节点名称
        master_name = "mymaster",
        -- 连接参数
        connect_timeout = 1000,  -- 1秒
        send_timeout = 1000,     -- 1秒
        read_timeout = 1000,     -- 1秒
        -- 认证
        password = "123456",
        -- 数据库
        db = 15,
    }
end

-- 创建Redis连接
local function connect_redis()
    local redis_connector = require "resty.redis.connector"
    local config = get_redis_config()
    
    local connector, err = redis_connector.new(config)
    if not connector then
        ngx.log(ngx.ERR, "Redis创建失败: ", err)
        return nil, err
    end
    
    local redis, err = connector:connect()
    if not redis then
        ngx.log(ngx.ERR, "Redis连接失败: ", err)
        return nil, err
    end
    
    return redis
end

-- 释放Redis连接
local function close_redis(redis)
    if not redis then
        return
    end
    
    local ok, err = redis:set_keepalive(10000, 100)
    if not ok then
        ngx.log(ngx.ERR, "释放Redis连接失败: ", err)
    end
end

-- 主流程
local auth_token = extract_auth_token()
if not auth_token then
    return ngx.redirect("http://www.xuzhihao.net")
end

local redis, err = connect_redis()
if not redis then
    return ngx.redirect("http://www.xuzhihao.net/500")
end

-- 使用pcall捕获可能的运行时错误
local ok, err = pcall(function()
    local res, query_err = redis:get(auth_token)
    if query_err then
        ngx.log(ngx.ERR, "Redis查询失败: ", query_err)
        error("redis_query_error")
    end
    
    if res == ngx.null then
        error("token_not_found")
    end
    
    -- 这里可以添加对查询结果的进一步处理
    -- 例如: ngx.ctx.user_info = cjson.decode(res)
end)

-- 确保连接被释放
close_redis(redis)

if not ok then
    if err == "token_not_found" then
        return ngx.redirect("http://www.xuzhihao.net")
    elseif err == "redis_query_error" then
        return ngx.redirect("http://www.xuzhihao.net/500")
    else
        ngx.log(ngx.ERR, "未知错误: ", err)
        return ngx.redirect("http://www.xuzhihao.net/500")
    end
end
```


3. 创建脚本current.lua

```lua
-- 从Cookie中提取auth_token
local function extract_auth_token()
    local cookie = ngx.var.http_cookie
    if not cookie then
        return nil
    end
    
    local m = ngx.re.match(cookie, "auth_token=([^;]+)", "jo")
    return m and m[1] or nil
end

-- Redis连接和操作封装
local function get_redis_connection()
    local redis = require "resty.redis"
    local red = redis:new()
    
    red:set_timeout(1000)  -- 1秒超时
    
    -- 连接Redis
    local ok, err = red:connect("172.17.17.161", 6379)
    if not ok then
        ngx.log(ngx.ERR, "Redis连接失败: ", err)
        return nil, err
    end
    
    -- 检查连接复用状态
    local reused, err = red:get_reused_times()
    if err then
        ngx.log(ngx.ERR, "获取连接复用状态失败: ", err)
        red:close()  -- 无法确定状态，直接关闭
        return nil, err
    end
    
    -- 全新连接需要认证
    if reused == 0 then
        local auth_ok, auth_err = red:auth("123456")
        if not auth_ok then
            ngx.log(ngx.ERR, "Redis认证失败: ", auth_err)
            red:close()
            return nil, auth_err
        end
    end
    
    -- 选择数据库
    local ok, err = red:select(1)
    if not ok then
        ngx.log(ngx.ERR, "Redis选择数据库失败: ", err)
        red:close()
        return nil, err
    end
    
    return red, nil
end

-- 释放Redis连接
local function close_redis(red)
    if not red then
        return
    end
    
    -- 尝试放回连接池
    local ok, err = red:set_keepalive(10000, 100)
    if not ok then
        ngx.log(ngx.ERR, "释放Redis连接失败: ", err)
        -- 连接池失败，直接关闭
        red:close()
    end
end

-- 主逻辑
local auth_token = extract_auth_token()
if not auth_token then
    ngx.status = ngx.HTTP_UNAUTHORIZED
    ngx.header.content_type = "text/plain"
    ngx.say("没有凭证")
    return ngx.exit(ngx.HTTP_UNAUTHORIZED)
end

-- 获取Redis连接
local red, err = get_redis_connection()
if not red then
    ngx.status = ngx.HTTP_INTERNAL_SERVER_ERROR
    ngx.say("系统错误")
    return ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
end

-- 查询Token
local res, err = red:get(auth_token)
if err then
    ngx.log(ngx.ERR, "Redis查询失败: ", err)
    close_redis(red)
    ngx.status = ngx.HTTP_INTERNAL_SERVER_ERROR
    ngx.say("系统错误")
    return ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
end

-- 处理查询结果
if res == ngx.null then
    close_redis(red)
    ngx.status = ngx.HTTP_UNAUTHORIZED
    ngx.say("凭证过期")
    return ngx.exit(ngx.HTTP_UNAUTHORIZED)
end

-- 成功返回结果
ngx.say(res)
close_redis(red)
```

#### 1.4.7 探测网站状态

下载并安装`lua-resty-http`库

```bash
wget https://github.com/ledgetech/lua-resty-http/archive/refs/tags/v0.17.2.tar.gz
tar -zxvf lua-resty-http-0.17.2.tar.gz
sudo cp -r lua-resty-http-0.17.2/lib/resty /usr/local/openresty/lualib/
```

1. 配置nginx.conf

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

2. 创建脚本check_health.lua

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

#### 1.4.8 限流

使用固定时间对url进行拦截，限制每秒最多请求30次，超出限制返回429状态码。

1. 配置nginx.conf

```nginx
server {
    listen 8080;
    charset utf-8;
    location / {			
        access_by_lua_file conf.d/token_bucket_limit.lua;
        proxy_pass http://backend/;
    }
}
```

2. 创建限流脚本token_bucket_limit.lua

```lua
-- Redis连接和操作封装
local function get_redis_connection()
    local redis = require "resty.redis"
    local red = redis:new()
    
    red:set_timeout(1000)  -- 1秒超时
    
    -- 连接Redis
    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.log(ngx.ERR, "Redis连接失败: ", err)
        return nil, err
    end
    
    -- 检查连接复用状态
    local reused, err = red:get_reused_times()
    if err then
        ngx.log(ngx.ERR, "获取连接复用状态失败: ", err)
        red:close()  -- 无法确定状态，直接关闭
        return nil, err
    end
    
    -- 全新连接需要认证
    if reused == 0 then
        local auth_ok, auth_err = red:auth("123456")
        if not auth_ok then
            ngx.log(ngx.ERR, "Redis认证失败: ", auth_err)
            red:close()
            return nil, auth_err
        end
    end
    
    -- 选择数据库
    local ok, err = red:select(0)
    if not ok then
        ngx.log(ngx.ERR, "Redis选择数据库失败: ", err)
        red:close()
        return nil, err
    end
    
    return red, nil
end

-- 释放Redis连接
local function close_redis(red)
    if not red then
        return
    end
    
    -- 尝试放回连接池
    local ok, err = red:set_keepalive(10000, 100)
    if not ok then
        ngx.log(ngx.ERR, "释放Redis连接失败: ", err)
        -- 连接池失败，直接关闭
        red:close()
    end
end

-- 获取Redis连接
local red, err = get_redis_connection()
if not red then
    ngx.status = ngx.HTTP_INTERNAL_SERVER_ERROR
    ngx.say("系统错误")
    return ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
end

-- 限流配置
local url = ngx.var.uri             -- 当前请求的 URL
local limit_per_period = 30         -- 每分钟30次
local period_seconds = 1            -- 时间窗口为1秒
local limit_key = "rate_limit:" .. url  -- Redis 键

-- 优化的Lua脚本（原子操作）
local script = [[
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local period = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])
    
    -- 获取当前计数和过期时间
    local current = tonumber(redis.call("GET", key)) or 0
    
    -- 检查是否超过限制
    if current >= limit then
        return 0  -- 超过限制
    end
    
    -- 增加计数
    redis.call("INCR", key)
    
    -- 如果是第一次设置，设置过期时间
    if current == 0 then
        redis.call("EXPIRE", key, period)
    end
    
    return 1  -- 允许通过
]]

-- 执行Lua脚本
local now = ngx.now()
local allowed, err = red:eval(script, 1, limit_key, limit_per_period, period_seconds, now)
if not allowed then
    ngx.log(ngx.ERR, "Failed to execute token bucket script: ", err)
    return ngx.exit(500)
end

close_redis(redis)

-- 处理限流结果
if allowed == 1 then
    ngx.log(ngx.INFO, "Request allowed: ", url)
else
    ngx.header["Retry-After"] = period_seconds  -- 建议客户端在时间窗口后重试
	ngx.header.content_type = "text/plain" 
	ngx.status = ngx.HTTP_TOO_MANY_REQUESTS
    ngx.say("Rate limit exceeded")
    ngx.exit(ngx.HTTP_TOO_MANY_REQUESTS)
end
```


#### 1.4.9 动态设置请求头

```nginx
server {
    listen 8080;
    charset utf-8;

    location / {
        access_by_lua_block {
            local auth_header = "Basic cm9vdDpBYTAwMDAwMA=="
            local uri = ngx.var.uri
            local args = ngx.req.get_uri_args()	
            if string.match(uri, "%.git/(git%-upload%-pack)/?$") then
                ngx.log(ngx.INFO, "====== 拉取匹配 ====== : " .. auth_header)
                ngx.req.set_header("Authorization", auth_header)
                return
            end
            if string.match(uri, "%.git/(git%-receive%-pack)/?$") then
                ngx.log(ngx.INFO, "====== 提交匹配 ====== : " .. auth_header)
                ngx.req.set_header("Authorization", auth_header)
                return
            end	
            if args.service and (args.service == "git-upload-pack" or args.service == "git-receive-pack") then
                ngx.log(ngx.INFO, "====== args匹配成功 ====== : " .. auth_header)
                ngx.req.set_header("Authorization", auth_header)
            end
        }        
        proxy_pass http://172.17.17.161:8080;
    }		
}
```


## 2. 模块

### 2.1 init_by_lua*

该指令在每次Nginx重新加载配置时执行，可以用来完成一些耗时模块的加载，或者初始化一些全局配置。

```lua
init_by_lua_block{
    redis = require "resty.redis"
    mysql = require "resty.mysql"
    cjson = require "cjson"
}
```

### 2.2 init_worker_by_lua*

该指令用于启动一些定时任务，如心跳检查、定时拉取服务器配置等。

### 2.3 set_by_lua*

该指令只要用来做变量赋值，这个指令一次只能返回一个值，并将结果赋值给Nginx中指定的变量。

### 2.4 rewrite_by_lua*

该指令用于执行内部URL重写或者外部重定向，典型的如伪静态化URL重写，本阶段在rewrite处理阶段的最后默认执行。

### 2.5 access_by_lua*

该指令用于访问控制、请求过滤、预处理等，不允许使用`ngx.say`和`ngx.print`。

**access_by_lua_file 可以放在 http、server、location 中，优先级依次递减，即 location > server > http**


```lua
access_by_lua_block {
    -- 检查用户是否登录（示例逻辑）
    local cookie = ngx.var.http_cookie
    local auth_token

    if cookie then
        -- 使用正则表达式匹配auth_token
        local m, err = ngx.re.match(cookie, "auth_token=([^;]+)", "jo")
        if m then
            auth_token = m[1]
        end
    end

    -- 如果未找到token，重定向到登录页面
    if not auth_token then
        -- 跳转到登录页面，保留原始请求URL用于登录后跳回
        local from_uri = ngx.var.request_uri
        local login_url = "/login?from=" .. ngx.escape_uri(from_uri)
        
        -- 执行302重定向
        return ngx.redirect(login_url)
    end
}
```

### 2.6 content_by_lua*

该指令是应用最多的指令，大部分任务是在这个阶段完成的，其他的过程往往为这个阶段准备数据，正式处理基本都在本阶段。

**content_by_lua_file 必须放在location内**

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

### 2.7 header_filter_by_lua*

该指令用于设置应答消息的头部信息。

### 2.8 body_filter_by_lua*

该指令是对响应数据进行过滤，如截断、替换。

### 2.9 log_by_lua*`

该指令用于在log请求处理阶段，用Lua代码处理日志，但并不替换原有log处理。

### 2.10 balancer_by_lua*

该指令主要的作用是用来实现上游服务器的负载均衡器算法

### 2.11 ssl_certificate_by_*

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
