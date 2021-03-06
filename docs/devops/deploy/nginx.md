# Nginx 1.20.2

## 1. 安装

http://nginx.org/

### 1.1 yum安装

#### 1.1.1 安装依赖

```bash
sudo yum install yum-utils
```

#### 1.1.2 编辑yum源文件

```bash
vim /etc/yum.repos.d/nginx.repo
# 添加内容
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```
默认情况使用稳定包，如果使用主线包执行下面命令，`第2位版本号`是偶数表示稳定包，基数表示开发包
```bash
sudo yum-config-manager --enable nginx-mainline
```

#### 1.1.3 进行安装

```bash
yum install -y nginx
```

### 1.2 源码安装

#### 1.2.1 下载

http://nginx.org/download/

```bash
cd /opt/software
wget http://nginx.org/download/nginx-1.20.2.tar.gz
```

#### 1.2.2 安装依赖

```bash
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

#### 1.2.3 添加用户

```bash
useradd nginx -s /sbin/nologin -M
```

#### 1.2.3 解压编译

```bash
cd /opt/software 
tar zxvf nginx-1.20.2.tar.gz -C /opt
cd /opt/nginx-1.20.2
./configure  --prefix=/usr/local/nginx --with-http_stub_status_module  --user=nginx --group=nginx --with-http_ssl_module
make && make install
```

#### 1.2.4 设置环境变量

```bash
vi /etc/profile
# 添加
PATH=$PATH:/usr/local/nginx/sbin
export PATH
# 使生效
source /etc/profile
```

#### 1.2.5 配置开机启动

在`/usr/lib/systemd/system`目录下添加nginx.service

```bash
vim /usr/lib/systemd/system/nginx.service

#添加内容
[Unit]
Description=nginx web service
Documentation=http://nginx.org/en/docs/
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=default.target
```

添加授权

```bash
chmod 755 /usr/lib/systemd/system/nginx.service
```

#### 1.2.6 启动服务

```bash
systemctl start nginx     # 启动
systemctl enable nginx    # 开机启动
nginx -V	# 查看版本

systemctl stop nginx      # 停止
systemctl restart nginx   # 重启
systemctl reload nginx    # 重新加载配置文件
systemctl status nginx    # 查看nginx状态
```

### 1.3 一键安装

```bash
#!/bin/bash

#关闭firewalld防火墙
systemctl stop firewalld
systemctl disable firewalld
systemctl mask firewalld

#关闭selinux
setenforce 0   #临时关闭
sed -i  '/^SELINUX/ s/enforcing/disabled/' /etc/selinux/config #永久关闭

#新建用户，用来启动nginx
useradd  -s /sbin/nologin nginx
#解决依赖关系，以及安装常用工具
yum install -y zlib zlib-devel openssl openssl-devel pcre pcre-devel gcc gcc-c++ automake autoconf make
yum install -y psmisc lsof net-tools vim wget

#下载nginx压缩包
mkdir -p /nginx
cd /nginx
curl  -O http://nginx.org/download/nginx-1.20.2.tar.gz

#解压文件
tar -zxvf  nginx-1.20.2.tar.gz 
cd nginx-1.20.2

#编译前的配置工作，根据实际需要选择常用模块
./configure --prefix=/usr/local/nginx --pid-path=/usr/local/nginx/logs/nginx.pid --user=nginx --group=nginx --build=scweb_server --with-threads  --with-file-aio --with-http_v2_module --with-http_ssl_module --with-stream --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module

#编译安装
make install

#添加到系统服务，设置开机自启
echo '[Unit]
Description=nginx web service
Documentation=http://nginx.org/en/docs/
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=default.target' > /usr/lib/systemd/system/nginx.service

chmod 755 /usr/lib/systemd/system/nginx.service

#延迟读取pid文件，避免提前读取pid文件（nginx启动需要时间）报错
mkdir -p /etc/systemd/system/nginx.service.d
printf "[Service]\nExecStartPost=/bin/sleep 0.1\n" > /etc/systemd/system/nginx.service.d/override.conf

#添加软链接和设置环境变量选其一
#ln -s /usr/local/nginx/sbin/nginx /usr/local/sbin/nginx
#将nginx配置到系统环境变量中
echo 'export PATH=$PATH:/usr/local/nginx/sbin' >> /etc/profile

#nginx配置文件高亮显示
echo 'syntax on' > ~/.vimrc
mkdir -p  ~/.vim/syntax
cd ~/.vim/syntax
wget -O nginx.vim http://www.vim.org/scripts/download_script.php?src_id=19394
echo "au BufRead,BufNewFile /usr/local/nginx/*,/usr/local/nginx/conf/* if &ft == '' | setfiletype nginx | endif " >> ~/.vim/filetype.vim

#日志轮转
echo '/usr/local/nginx/logs/*.log {
    create 0640 nginx root
    daily
    rotate 10
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /usr/local/nginx/logs/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
' > /etc/logrotate.d/nginx

systemctl daemon-reload
systemctl enable nginx  #开机自启
systemctl status nginx
```

```bash
sed -i 's/\r$//' run.sh  
```

### 1.4 卸载

```bash
yum remove -y nginx					# yum卸载
find / -name nginx | xargs rm -rf	# 编译删除
```


## 2. 命令

### 2.1 信号控制

| 信号     | 作用                                                       |
| -------- | ---------------------------------------------------------- |
| TERM/INT | 立即关闭整个服务                                           |
| QUIT     | "优雅"地关闭整个服务                                       |
| HUP      | 重读配置文件并使用服务对新配置项生效                       |
| USR1     | 重新打开日志文件，可以用来进行日志切割                     |
| USR2     | 平滑升级到最新版的nginx                                    |
| WINCH    | 所有子进程不在接收处理新连接，相当于给work进程发送QUIT指令 |

### 2.2 命令行控制

```lua
-?和-h:显示帮助信息
-v:打印版本号信息并退出
-V:打印版本号信息和配置信息并退出
-t:测试nginx的配置文件语法是否正确并退出
-T:测试nginx的配置文件语法是否正确并列出用到的配置文件信息然后退出
-q:在配置测试期间禁止显示非错误消息
-s:signal信号，后面可以跟 ：
    stop[快速关闭，类似于TERM/INT信号的作用]
    quit[优雅的关闭，类似于QUIT信号的作用] 
    reopen[重新打开日志文件类似于USR1信号的作用] 
    reload[类似于HUP信号的作用]
-p:prefix，指定Nginx的prefix路径，(默认为: /usr/local/nginx/)
-c:filename,指定Nginx的配置文件路径,(默认为: conf/nginx.conf)
-g:用来补充Nginx配置文件，向Nginx服务指定启动时应用全局的配置
```

### 2.2 不停机升级

#### 2.2.1 使用信号

1. 备份

```bash
cd /usr/local/nginx/sbin
mv nginx nginxold
```

2. 新程序拷贝到原安装路径下

```bash
cd ~/nginx/core/nginx-1.16.1/objs
cp nginx /usr/local/nginx/sbin
```

3. 发送USR2信号给master进程，告诉master进程要平滑升级，这个时候，会重新开启对应的master进程和work进程，整个系统中将会有两个master进程，并且新的master进程的PID会被记录在`/usr/local/nginx/logs/nginx.pid`而之前的旧的master进程PID会被记录

```bash
kill -USR2 PID / kill -USR2 `cat /usr/local/nginx/logs/nginx.pid`
```

4. 发送信号QUIT给旧版本退出

```
kill -QUIT `more /usr/local/logs/nginx.pid.oldbin`
```

#### 2.2.2 使用makefile命令

1. 备份

```bash
cd /usr/local/nginx/sbin
mv nginx nginxold
```

2. 新程序拷贝到原安装路径下

```bash
cd ~/nginx/core/nginx-1.16.1/objs
cp nginx /usr/local/nginx/sbin
```

3. 升级

```bash
cd ~/nginx/core/nginx-1.16.1
make upgrade
```


## 3. 高级

### 3.1 配置文件

```bash
./configure --prefix=/usr/local/nginx --with-stream --with-http_stub_status_module --with-http_ssl_module
```

#### 3.1.1 nginx.conf

```conf
user nginx;
worker_processes 32;
error_log /var/log/nginx/nginx_error.log crit;
pid /var/run/nginx.pid;
worker_rlimit_nofile 102400;
events {
  use epoll;
  worker_connections 102400;
}

http {
  include mime.types;
  default_type application/octet-stream;
  # 自定义日志
  log_format access '$host - $server_addr $remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$remote_addr" "$http_x_forwarded_for" "$proxy_add_x_forwarded_for" "$http_x_real_ip" "$proxy_add_x_forwarded_for" "$upstream_addr"'; 
  log_format post '$host - $remote_addr\t$remote_user\t[$time_local]\t"$request"\t$status\t$bytes_sent\t';
  log_format ip '$host - $remote_addr - $remote_user [$time_local] "$request" - [$upstream_addr] - [$upstream_status]';
  log_format rc escape=json '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" $request_body';
  access_log  /var/log/nginx/ip_access.log  post;
  # http参数
  proxy_headers_hash_max_size  51200;   # 头部哈希表的最大值
  proxy_headers_hash_bucket_size 6400;  # 头部哈希表大小
  server_names_hash_bucket_size 512;    # 服务器名字的哈希表大小
  client_header_buffer_size 128k;       # 客户端请求头buffer大小
  client_body_buffer_size 8m            # 请求主体缓冲区
  large_client_header_buffers 4 128k;   # 客户请求头缓冲大小
  # 静态资源优化
  sendfile on;                          # 开启高效的文件传输模式
  tcp_nopush on;                        # 提升网络传输效率
  tcp_nodelay on;                       # 提高网络包传输
  keepalive_timeout 60s;                # 开启TCP连接保持
  # 资源压缩优化
  gzip on;
  gzip_types text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml text/javascript application/javascript application/x-javascript text/x-json application/json application/x-web-app-manifest+json text/css text/plain text/x-component font/opentype application/x-font-ttf application/vnd.ms-fontobject image/x-icon;   # 压缩文件类型
  client_max_body_size 256m;            # 最大上传限制
  gzip_vary on;                         # 压缩发送携带响应头
  gzip_buffers 4 16k;                   # 缓存空间大小
  gzip_disable "MSIE [1-6]\.";          # 关闭压缩的排除条件
  gzip_http_version 1.1;                # 使用Gzip的HTTP最低版本
  gzip_min_length 4k;                   # 关闭压缩传输数据大小限制/最小压缩限制
  gzip_comp_level 3;                    # 压缩率
  ##
  open_file_cache max=102400 inactive=20s;
  open_file_cache_valid 30s;
  open_file_cache_min_uses 1;
  server_tokens off;
  ##
  proxy_connect_timeout 30s;
  proxy_send_timeout 150s;
  proxy_read_timeout 150s;
  ## 反向代理优化
  proxy_buffering on;
  proxy_buffer_size 1024k;
  proxy_buffers 4 1024k;
  proxy_busy_buffers_size 1024k;
  proxy_temp_file_write_size 1024k;
  
  # 加载其他配置
  include /usr/local/openresty/nginx/conf/blob.xuzhihao.net.conf;
  # 负载均衡
  include /usr/local/openresty/nginx/conf/blob.xuzhihao.net_upstream.conf;
}
```

#### 3.1.2 blob.xuzhihao.net_upstream.conf

```conf
upstream blob.xuzhihao.net {
  ip_hash;
  server 192.168.3.200:7535;
  server 192.168.3.201:7535;
}
```

#### 3.1.3 blob.xuzhihao.net.conf

```conf
server {
		listen 80;
		server_name blob.xuzhihao.net;
		rewrite ^(.*)$ https://$host$1 permanent;
}
server {
		listen 443 ssl;
		server_name blob.xuzhihao.net;
		charset utf-8;
		ssl_certificate /usr/local/nginx/conf/sx/sxgw.vjspnet.cn_chain.crt;
		ssl_certificate_key /usr/local/nginx/conf/sx/sxgw.vjspnet.cn_key.key;
		ssl_session_cache shared:SSL:20m;
        ssl_session_timeout 5m;
		ssl_buffer_size 256k;
		ssl_session_tickets on;
		ssl_stapling on;
		ssl_stapling_verify on;
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
		ssl_ciphers HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM;
		ssl_prefer_server_ciphers on;
        resolver 223.5.5.5 114.114.114.114 180.76.76.76 valid=300s;
		resolver_timeout 10s;
		index index.html index.htm index.jsp index.do index.action;
		access_log /usr/local/nginx/conf/sx/logs/sxgw.log;
		location / {
				proxy_redirect off;
				proxy_set_header Host $host;
				proxy_set_header X-Real-IP $remote_addr;
				proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
				add_header Access-Control-Allow-Origin *;
				add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
				add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';
				proxy_pass http://blob.xuzhihao.net;
		}
}
```

### 3.2 负载均衡upstream

#### 3.2.1 负载均衡状态

```conf
upstream backend{
	server 192.168.3.200:9001 down;
	server 192.168.3.201:9002 backup;
	server 192.168.3.202:9003 max_fails=3 fail_timeout=15;
}
```

#### 3.2.2 负载均衡策略

1. weight

```conf
upstream backend{
	server 192.168.200.146:9001 weight=10;
	server 192.168.200.146:9002 weight=5;
	server 192.168.200.146:9003 weight=3;
}
```

2. ip_hash

```conf
upstream blob.xuzhihao.net {
  ip_hash;
  server 192.168.3.200:7535;
  server 192.168.3.201:7535;
}
```

3. least_conn

```
upstream backend{
	least_conn;
	server 192.168.200.146:9001;
	server 192.168.200.146:9002;
	server 192.168.200.146:9003;
}
```

4. url_hash

```conf
upstream backend{
	hash &request_uri;
	server 192.168.200.146:9001;
	server 192.168.200.146:9002;
	server 192.168.200.146:9003;
}
```

5. fair

```conf
upstream backend{
	fair;
	server 192.168.200.146:9001;
	server 192.168.200.146:9002;
	server 192.168.200.146:9003;
}
```


### 3.3 四层负载stream

```bash
./configure --prefix=/usr/local/nginx --with-stream
```

```conf
stream {
    upstream back{
            server 192.168.3.200:3306 up;
            server 192.168.3.201:3306 up;
    }
    server {
            listen 3301;
            proxy_connect_timeout 5s;
            proxy_timeout 300s;
            proxy_pass back;
    }
}
```

### 3.4 泛域名

```bash
./configure  --prefix=/usr/local/nginx --with-http_sub_module
```

```conf
server
{ 
    listen 80;
	server_name  ~^(?<serno>.+).xuzhihao.net$;
	server_name_in_redirect off; 
	location / {
		rewrite ^(.*)$ /$serno$1 break;
		#proxy_pass http://127.0.0.1:8080;
		root D:/workspace/;
		proxy_set_header   Host    $host;
		proxy_set_header   X-Real-IP   $remote_addr;
		proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
	}
	access_log logs/web.log;
}
```


### 3.5 二级路径

```conf
server {
    listen 80;
    server_name  www.xuzhihao.net;
    charset utf-8; 
    index index.html;
    location / {
        root /home/project;
    }
    location /emoji/ {
        alias /home/project/;   # 必须以斜线结尾，实际路径/home/project/emoji/
    }
    location /blob/ {
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://192.168.3.200:8000/blob/;
    }
}
```

### 3.6 错误页

#### 3.6.1 指定具体跳转地址

```bash
server {
	error_page 404 http://www.xuzhihao.net;
}
```

#### 3.6.2 指定重定向地址

```bash
server{
	error_page 404 /50x.html;
	error_page 500 502 503 504 /50x.html;
	location =/50x.html{
		root html;
	}
}
```

#### 3.6.3 使用location的@符号

```bash
server{
	error_page 404 @jump_to_error;
	location @jump_to_error {
		default_type text/plain;
		return 404 'Not Found Page...';
	}
}
```

#### 3.6.4 修改指定状态码

```bash
server{
	error_page 404 =200 /50x.html;
	location =/50x.html{
		root html;
	}
}
```

### 3.7 跨域

```conf
location /getUser {
	default_type application/json;
	add_header Content-Type 'text/html; charset=utf-8';
	add_header Access-Control-Allow-Origin *;
	add_header Access-Control-Allow-Methods GET,POST,PUT,DELETE;
	if ( $query_string ~* ^(.*)v=1.0$ ){
		return 200 '{"id":1,"name":"我是个大盗贼","age":29}';
	}
	if ( $query_string ~* ^(.*)v=2.0$ ){
		return 200 '{"id":1,"name":"我们都是好孩子","age":29}';
	}
	return 200 '{"id":3,"name":"小牛心心","age":29}';
}
```

```conf
location /postUser {
	default_type application/json;
	add_header Content-Type 'text/html; charset=utf-8';
	if ($request_method !~* POST) {
		return 403;
	}
	return 200 '{"id":3,"name":"小牛心心","age":29}';
}
```

### 3.8 防盗链

```conf
location ~*\.(png|jpg|gif){
    valid_referers none blocked www.hwcq.online 192.168.3.200 *.xuzhihao.net xuzhihao.*  ~\.hwcq\.;
    if ($invalid_referer){
        return 403;
    }
    root /usr/local/nginx/html;
}
```

```conf
location /images {
    valid_referers none blocked www.hwcq.online 192.168.3.200 *.xuzhihao.net xuzhihao.*  ~\.hwcq\.;
    if ($invalid_referer){
        rewrite ^/ http://www.web.com/images/forbidden.png;
    }
    root /usr/local/nginx/html;
}
```

### 3.9 Rewrite

常用变量

| 变量               | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| $args              | 变量中存放了请求URL中的请求指令。比如http://192.168.200.133:8080?arg1=value1&args2=value2中的"arg1=value1&arg2=value2"，功能和$query_string一样 |
| $http_user_agent   | 变量存储的是用户访问服务的代理信息(如果通过浏览器访问，记录的是浏览器的相关版本信息) |
| $host              | 变量存储的是访问服务器的server_name值                        |
| $document_uri      | 变量存储的是当前访问地址的URI。比如http://192.168.200.133/server?id=10&name=zhangsan中的"/server"，功能和$uri一样 |
| $document_root     | 变量存储的是当前请求对应location的root值，如果未设置，默认指向Nginx自带html目录所在位置 |
| $content_length    | 变量存储的是请求头中的Content-Length的值                     |
| $content_type      | 变量存储的是请求头中的Content-Type的值                       |
| $http_cookie       | 变量存储的是客户端的cookie信息，可以通过add_header Set-Cookie 'cookieName=cookieValue'来添加cookie数据 |
| $limit_rate        | 变量中存储的是Nginx服务器对网络连接速率的限制，也就是Nginx配置中对limit_rate指令设置的值，默认是0，不限制。 |
| $remote_addr       | 变量中存储的是客户端的IP地址                                 |
| $remote_port       | 变量中存储了客户端与服务端建立连接的端口号                   |
| $remote_user       | 变量中存储了客户端的用户名，需要有认证模块才能获取           |
| $scheme            | 变量中存储了访问协议                                         |
| $server_addr       | 变量中存储了服务端的地址                                     |
| $server_name       | 变量中存储了客户端请求到达的服务器的名称                     |
| $server_port       | 变量中存储了客户端请求到达服务器的端口号                     |
| $server_protocol   | 变量中存储了客户端请求协议的版本，比如"HTTP/1.1"             |
| $request_body_file | 变量中存储了发给后端服务器的本地文件资源的名称               |
| $request_method    | 变量中存储了客户端的请求方式，比如"GET","POST"等             |
| $request_filename  | 变量中存储了当前请求的资源文件的路径名                       |
| $request_uri       | 变量中存储了当前请求的URI，并且携带请求参数，比如http://192.168.200.133/server?id=10&name=zhangsan中的"/server?id=10&name=zhangsan" |

目录合并

```conf
server {
	listen 80;
	server_name www.web.name;
	location /server{
		rewrite ^/server-([0-9]+)-([0-9]+)-([0-9]+)-([0-9]+)\.html$ /server/$1/$2/$3/$4/$5.html last;
	}
}
```

### 3.10 web缓存

```conf
http{
	proxy_cache_path /usr/local/proxy_cache levels=2:1 keys_zone=xzh:200m inactive=1d max_size=20g; # 缓存文件的存放路径
	upstream backend{
		server 192.168.3.200:8080;
	}
	server {
		listen       8080;
        server_name  localhost;
        location / {
            if ($request_uri ~ /.*\.js$){
                set $nocache 1;
            }
        	proxy_cache xzh;                    # 和缓存区名称保存一致
            proxy_cache_key xuzhihao;           # 缓存的key值
            # proxy_cache_key $scheme$proxy_host$request_uri;   
            proxy_cache_min_uses 10;            # 10次以后缓存
            proxy_cache_valid 200 5d;           # 不同响应状态码缓存时长
            proxy_cache_valid 404 30s;
            proxy_cache_valid any 1m;
            add_header nginx-cache "$upstream_cache_status";  # 缓存状态
            proxy_no_cache $nocache $cookie_nocache $arg_nocache $arg_comment;  # 不缓存条件
            proxy_cache_bypass $nocache $cookie_nocache $arg_nocache $arg_comment;  # 从服务器获取并且缓存？？？没明白意义何在
        	proxy_pass http://backend/js/;
        }
	}
}
```

ngx_cache_purge缓存清除

```conf
# 下载并上传
ngx_cache_purge-2.3.tar.gz
# 对资源文件进行解压缩
tar -zxf ngx_cache_purge-2.3.tar.gz
# 修改文件夹名称，方便后期配置
mv ngx_cache_purge-2.3 purge
# 查询Nginx的配置参数
nginx -V
# 进入Nginx的安装目录，使用./configure进行参数配置
./configure --add-module=/root/nginx/module/purge
# 使用make进行编译
make
# 将nginx安装目录的nginx二级制可执行文件备份
mv /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginxold
# 将编译后的objs中的nginx拷贝到nginx的sbin目录下
cp objs/nginx /usr/local/nginx/sbin
# 使用make进行升级
make upgrade
```

在nginx配置文件中进行如下配置

```conf
server{
	location ~ /purge(/.*) {
		proxy_cache_purge xzh xuzhihao;
	}
}
```

### 3.11 目录索引

#### 3.11.1 下载插件

```bash
cd /opt/software
wget https://codeload.github.com/aperezdc/ngx-fancyindex/zip/master -O ngx-fancyindex-master.zip    # 插件

```

#### 3.11.2 解压编译

```bash
unzip ngx-fancyindex-master.zip
mv ngx-fancyindex-master /opt
cd nginx-1.20.2
./configure  --prefix=/usr/local/nginx --with-http_stub_status_module  --user=nginx --group=nginx --with-http_ssl_module --add-module=../ngx-fancyindex-master
make  # 不要 make install
cp objs/nginx  /usr/local/nginx/sbin/  # 复制重新编译的nginx文件到nginx原来安装目录下
```

#### 3.11.3 配置模板


1. 下载模板

```bash
wget https://codeload.github.com/xzh-net/nginx-fancyindex/zip/refs/heads/main -O nginx-fancyindex-main.zip -P /home/www  # 模板
cd /home/www
unzip nginx-fancyindex.zip	
mv nginx-fancyindex fancyindex
```

2. 配置fancyindex路径

?> 注意：/home/www是默认站点根目录，fancyindex文件夹在`/home/www`下

```conf
vi /usr/local/nginx/conf/nginx.conf

# 修改内容
server 
{
  listen 80;
  server_name www.hwcq.online;
  charset utf-8; 
  location /	{
      root /home/www/; 
      fancyindex on;                                 # 开启nginx目录浏览功能 
      fancyindex_localtime on;                       # 显示文件修改时间为服务器本地时间 
      fancyindex_exact_size off;                     # 文件大小从KB开始显示 
      fancyindex_header "/fancyindex/header.html";   # 设置footer为当前目录下的footer.html
      fancyindex_footer "/fancyindex/footer.html";   # 设置footer为当前目录下的footer.html
      fancyindex_ignore "fancyindex";                # 忽略
      fancyindex_ignore "cgi";                       # 忽略
      fancyindex_ignore ".php";                      # 忽略
      fancyindex_time_format "%Y-%m-%d %H:%M:%S";
      fancyindex_name_length 200;
  }
}
```

#### 3.11.4 md在线预览

?> 注意：/home/www/public是默认站点根目录，应用public和fancyindex文件夹是平行关系

1. nginx配置

```conf
http {  
    include mime.types;
    default_type application/octet-stream;

    client_max_body_size 100M;
    charset utf-8;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    '$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log off;

    server_tokens   off;
    sendfile        on; 

    keepalive_timeout  65;

    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    client_header_buffer_size 64k;
    large_client_header_buffers 4 64k;
    client_body_buffer_size 20m;

    index index.html;

    include /usr/local/nginx/sites-enabled/*;
}
```

2. 站点配置

```bash
mkdir -p /usr/local/nginx/sites-enabled
cd /usr/local/nginx/sites-enabled
vi default
```

```conf
server {
    listen   80; ## listen for ipv4; this line is default and implied
    listen   [::]:80 ipv6only=on; ## listen for ipv6

    listen   443 ssl;
    listen   [::]:443 default ipv6only=on ssl;

    # Enable SSL
    ssl_certificate /home/www/ssl/ssl.crt;
    ssl_certificate_key /home/www/ssl/ssl.key;
    ssl_session_timeout 10m;
    ssl_session_cache   shared:SSL:10m;
    ssl_protocols TLSv1.1 TLSv1.2;
    ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv3:+EXP;
    ssl_prefer_server_ciphers on;

    root /home/www/public;
    index index.html index.htm;

    location / {
        auth_basic "off";
        include /home/www/fancyindex/fancyindex.conf;
    }

    location ~ \.(?:md|markdown)$$ {
        auth_basic "off";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8002; # Markdown Renderer server.
    }

    location =/assets/gfm.css {
        auth_basic "off";
        proxy_pass http://127.0.0.1:8002; # Markdown Renderer server.
    }

    #fixup fancyindex subrequest
    location ^~ /fancyindex/ {
        auth_basic "off";
        alias /home/www/fancyindex/;
    }

    #fixup favicon.ico
    location /favicon.ico {
        alias /home/www/fancyindex/meta/favicon.ico;
    }

    location =/fancyindex/fancyindex.conf {
        deny  all;
    }

    location =/fancyindex/README.md {
        deny  all;
    }

    location =passwd {
        deny  all;
    }

  add_header X-Frame-Options "SAMEORIGIN";
  add_header X-XSS-Protection "1; mode=block";
  add_header X-Content-Type-Options "nosniff";

  location = /favicon.ico { access_log off; log_not_found off; }
  location = /robots.txt  { access_log off; log_not_found off; }

  location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc|ttf|ttc|otf|eot|woff)$ {
    auth_basic "off";
    expires max;
    access_log off;
    add_header Pragma public;
    add_header Cache-Control "public, must-revalidate, proxy-revalidate";
  }

  location ~* \.(?:css|js)$ {
    auth_basic "off";
    expires 1y;
    add_header Cache-Control "public";
  }

  # deny access to . files, for security
  location ~ /\.(?!well-known).* {
    access_log off;
    log_not_found off;
    deny all;
  }

  location ~* (?:\.(?:bak|config|db|sql|fla|psd|ini|log|sh|inc|swp|dist)|~)$ {
    deny all;
    access_log off;
    log_not_found off;
  }
}
```

#### 3.11.5 启动预览服务器

```
/usr/local/bin/markdown-renderer -mode local -root /home/www/public/
```

### 3.12 IP过滤

1. 屏蔽ip

单独网站屏蔽IP的方法，把include blocksip.conf放到网址对应的在server{}语句块，所有网站屏蔽IP的方法，把include blocksip.conf放到http {}语句块。

```bash
include blockip.conf; 
# 添加内容
allow  180.164.67.212;
allow  59.46.186.253;
deny all;

http {
    # include blockip.conf;
    server {
        listen       28101;
        server_name  _;
        # include blockip.conf;
        location /  {
                proxy_redirect off;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_pass http://127.0.0.1:18101;
                # include blockip.conf;
        }
        
    }
}
```

统计ip访问次数

```bash
awk '{print $1}' /var/log/nginx/access.log |sort |uniq -c|sort -n
```

2. 通过User-Agent过滤爬虫

```bash
location / {
    if ($http_user_agent ~* "scrapy|python|curl|java|wget|httpclient|okhttp") {
        return 503;
    }
}
```

## 4. SSL证书

### 4.1 openssl自签名

```bash
mkdir /root/cert
cd /root/cert
openssl genrsa -des3 -out server.key 1024           # 创建SSL证书私钥
openssl req -new -key server.key -out server.csr    # 创建SSL证书签名请求文件
cp server.key server.key.org        
openssl rsa -in server.key.org -out server.key      # 利用私钥生成一个不需要输入密码的密钥文件
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt  # 生成SSL证书，有效期为365天
```

```conf
server {
    listen       80; # 同时支持HTTP
    listen       443 ssl; # 添加HTTPS支持
    server_name  api.xuzhihao.net; #修改域名

    #ssl配置
    ssl_certificate      /usr/share/nginx/html/ssl/api/api.crt; # 配置证书
    ssl_certificate_key  /usr/share/nginx/html/ssl/api/api_nopass.key; # 配置证书私钥
    ssl_protocols        TLSv1 TLSv1.1 TLSv1.2; # 配置SSL协议版本 # 配置SSL加密算法
    ssl_ciphers          ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers  on; # 优先采取服务器算法
    ssl_session_cache    shared:SSL:10m; # 配置共享会话缓存大小
    ssl_session_timeout  10m; # 配置会话超时时间

    location / {
        proxy_pass   http://192.168.3.101:8080;         # 设置代理服务访问地址
        proxy_set_header  Host $http_host;              # 设置客户端真实的域名（包括端口号）
        proxy_set_header  X-Real-IP  $remote_addr;      # 设置客户端真实IP
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for; # 设置在多层代理时会包含真实客户端及中间每个代理服务器的IP
        proxy_set_header X-Forwarded-Proto $scheme;     # 设置客户端真实的协议（http还是https）
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

```bash
openssl genrsa -des3 -out server.key 2048
openssl req -new -x509 -key server.key -out server.crt -days 3650
```

### 4.2 阿里云

- 阿里云上可以申请的免费证书目前只有支持单个域名的DV级SSL证书。比如说你有`blog.xuzhihao.net`和`api.xuzhihao.net`两个二级域名需要使用HTTPS，就需要申请两个SSL证书。

![](../../assets/_images/devops/deploy/nginx/nginx_https_start_05.png)

```
#SSL配置
ssl_certificate      /usr/share/nginx/html/ssl/blog/blog.xuzhihao.net.pem; # 配置证书
ssl_certificate_key  /usr/share/nginx/html/ssl/blog/blog.xuzhihao.net.key; # 配置证书私钥
```

### 4.3 FreeSSL

https://freessl.cn/ 选择双域名离线生成

![](../../assets/_images/devops/deploy/nginx/nginx_https_start_08.png)


### 4.4 自动申请

- `acme.sh`脚本实现了`acme`协议, 可以从`letsencrypt`生成免费的证书。一般我们申请的证书有效期都是1年，过期就要重新申请了，使用`acme.sh`脚本可以实现到期自动申请，再也不用担心证书过期了！

![](../../assets/_images/devops/deploy/nginx/nginx_https_start_09.png)

- 附上官网地址：https://github.com/acmesh-official/acme.sh


## 5. OpenRestry

https://openresty.org

### 5.1 安装

```bash
wget https://openresty.org/download/openresty-1.15.8.2.tar.gz
tar -zxf openresty-1.15.8.2.tar.gz
cd openresty-1.15.8.2
./configure
make && make install
cd /usr/local/openresty/nginx/
```

vi nginx.conf
```conf
location /lua{
    default_type 'text/html';
    content_by_lua 'ngx.say("<h1>HELLO,OpenRestry</h1>")';
}
```

### 5.2 模块指令

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
location /{
			default_type "text/html";
			content_by_lua_block{
				
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
