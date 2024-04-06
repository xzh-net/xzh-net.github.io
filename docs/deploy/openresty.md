# OpenResty 1.25

- 官方网站：https://openresty.org

## 1. 安装

### 1.1 下载

```bash
cd /opt/software
wget https://openresty.org/download/openresty-1.25.3.1.tar.gz
```

### 1.2 安装依赖

```bash
yum install pcre-devel openssl-devel gcc curl zlib-devel readline-devel
```

### 1.3 解压编译

```bash
cd /opt/software
tar -zxf openresty-1.25.3.1.tar.gz
cd openresty-1.25.3.1
./configure
make && make install
```

### 1.4 设置环境变量

```bash
vi /etc/profile
export PATH=$PATH:/usr/local/openresty/nginx/sbin
source /etc/profile
```

### 1.5 入门案例

- 组件地址：https://openresty.org/en/lua-resty-redis-library.html

1. 入门案例

```bash
cd /usr/local/openresty/nginx/conf
vi nginx.conf
```

```conf
location / {
    default_type 'text/html';
    content_by_lua_block {
        ngx.say("hello,OpenRestry");
    }
}
```

也可以使用`content_by_lua_file`来引入一个lua文件

```conf
location / {
    default_type 'text/html';
    content_by_lua_file /usr/local/openresty/nginx/conf/test.lua;
}
```


2. 获取http请求信息

```lua
-- 获取get请求参数
local arg = ngx.req.get_uri_args()
for k,v in pairs(arg) do
    ngx.say("[GET] key:", k, " v:", v)
    ngx.say("<br>")
end

-- 获取post请求参数
ngx.req.read_body()
local arg = ngx.req.get_post_args()
for k,v in pairs(arg) do
    ngx.say("[POST] key:", k, " v:", v)
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


3. 操作redis

```lua
local redis = require "resty.redis"
local red = redis:new()

red:set_timeout(1000) -- 1 sec

local ok, err = red:connect("192.168.2.201", 6379)
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


ok, err = red:set("xzh", "我们都是好孩子")
if not ok then
	ngx.say("failed to set xzh: ", err)
	return
end

ngx.say("set result: ", ok)

local res, err = red:get("xzh")
if not res then
	ngx.say("failed to get xzh: ", err)
	return
end

if res == ngx.null then
	ngx.say("xzh not found.")
	return
end

ngx.say("xzh: ", res)

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


## 3. Twemproxy

- 官方网站：https://github.com/twitter/twemproxy/

### 3.1 安装autoconf

```bash
cd /opt/software
wget http://ftp.gnu.org/gnu/autoconf/autoconf-2.69.tar.gz
tar zxvf autoconf-2.69.tar.gz
cd autoconf-2.69
./configure --prefix=/usr/local
make && make install
autoconf --version
```

> 编译过程提示报错 `checking for GNU M4 that supports accurate traces... configure: error: no acceptable m4 could be found in $PATH.` 需要安装m4

```bash
cd /opt/software
wget http://mirrors.kernel.org/gnu/m4/m4-1.4.18.tar.gz
tar -xvf m4-1.4.18.tar.gz
cd m4-1.4.18
./configure
make && make install
m4 --version
```

> 编译过程提示报错 `BEGIN failed--compilation aborted at ../lib/Autom4te/C4che.pm line 33` 需要安装perl的依赖包 `perl-Data-Dumper`

包地址：http://rpmfind.net/linux/rpm2html/search.php?query=perl-Data-Dumper(x86-64)

```bash
cd /opt/software
wget http://rpmfind.net/linux/centos/7.9.2009/os/x86_64/Packages/perl-Data-Dumper-2.145-3.el7.x86_64.rpm
rpm -ivh perl-Data-Dumper-2.145-3.el7.x86_64.rpm
```

### 3.2 安装automake

```bash
cd /opt/software
wget http://ftp.gnu.org/gnu/automake/automake-1.15.tar.gz
tar -zvxf automake-1.15.tar.gz
cd automake-1.15
./configure
make && make install
aclocal --version
```

> 编译过程提示报错 `Try --no-discard-stderr' if option outputs to stderr` 修改源码 `automake-1.15/Makefile.in`

```bash
# 添加--no-discard-stderr选项
doc/automake-$(APIVERSION).1: $(automake_script) lib/Automake/Config.pm
	$(update_mans) automake-$(APIVERSION) --no-discard-stderr
```

### 3.3 安装libtool

```bash
cd /opt/software
wget https://ftp.gnu.org/gnu/libtool/libtool-2.4.6.tar.gz
tar -zvxf libtool-2.4.6.tar.gz
cd libtool-2.4.6
./configure
make && make install
libtool --version
```

### 3.4 编译Twemproxy

```bash
cd /opt/software
yum install perl-Thread-Queue
wget https://github.com/twitter/twemproxy/releases/download/0.5.0/twemproxy-0.5.0.tar.gz
tar -zvxf twemproxy-0.5.0.tar.gz
cd twemproxy-0.5.0
autoreconf -fvi 
./configure --prefix=/usr/local/twemproxy/
make && make install
```

### 3.5 配置Twemproxy

```bash
cd /opt/software/twemproxy-0.5.0
cp -r ./conf /usr/local/twemproxy/
cd /usr/local/twemproxy/conf
vi nutcracker.yml
```

```yml
alpha:
  listen: 127.0.0.1:22121
  hash: fnv1a_64
  distribution: ketama
  auto_eject_hosts: true
  redis: true
  redis_auth: 123456
  server_retry_timeout: 2000
  server_failure_limit: 1
  servers:
   - 192.168.2.201:6379:1
```

### 3.6 启动服务

设置环境变量

```bash
echo "PATH=$PATH:/usr/local/twemproxy/sbin/" >> /etc/profile
source /etc/profile
```

启动服务

```bash
nutcracker
```

### 3.7 客户端测试

```bash
redis-cli -h 192.168.2.201 -p 22121
```

