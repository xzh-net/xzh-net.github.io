# Nginx 1.22.1

- 官方网站：http://nginx.org

## 1. 安装

### 1.1 编译安装

#### 1.1.1 下载

```bash
cd /opt/software
wget http://nginx.org/download/nginx-1.22.1.tar.gz
```

#### 1.1.2 安装依赖

```bash
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

#### 1.1.3 添加用户

```bash
useradd nginx -s /sbin/nologin -M
```

#### 1.1.4 解压编译

```bash
cd /opt/software 
tar zxvf nginx-1.22.1.tar.gz
cd nginx-1.22.1

./configure --prefix=/usr/local/nginx --pid-path=/usr/local/nginx/logs/nginx.pid \
--user=nginx --group=nginx --build=web_server --with-pcre --with-compat --with-file-aio \
--with-stream --with-stream_ssl_module \
--with-http_ssl_module --with-http_realip_module \
--with-http_sub_module \
--with-mail_ssl_module \
--with-http_stub_status_module --with-http_gzip_static_module \

make && make install
```

#### 1.1.5 设置环境变量

```bash
vi /etc/profile
# 添加
PATH=$PATH:/usr/local/nginx/sbin
export PATH
# 使生效
source /etc/profile
```

#### 1.1.6 配置开机启动

在`/usr/lib/systemd/system`目录下添加nginx.service

```bash
vim /usr/lib/systemd/system/nginx.service
```

添加内容
```conf
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
systemctl daemon-reload
```

#### 1.1.7 启动服务

```bash
nginx -V	# 查看版本
systemctl enable nginx    # 开机启动
systemctl start nginx     # 启动
systemctl stop nginx      # 停止
systemctl restart nginx   # 重启
systemctl reload nginx    # 重新加载配置文件
systemctl status nginx    # 查看状态
```

#### 1.1.8 日志切割

```bash
echo '/usr/local/nginx/logs/*.log {   # 可以指定多个路径
    daily                      # 日志轮询周期，weekly,monthly,yearly
    rotate 30                  # 保存30天数据，超过的则删除
    size +100M                 # 超过100M时分割，单位K,M,G，优先级高于daily
    compress                   # 切割后压缩，也可以为nocompress
    delaycompress              # 切割时对上次的日志文件进行压缩
    dateext                    # 日志文件切割时添加日期后缀
    missingok                  # 如果没有日志文件也不报错
    notifempty                 # 日志为空时不进行切换，默认为ifempty
    create 640 nginx nginx     # 使用该模式创建日志文件
    sharedscripts              # 所有的文件切割之后只执行一次下面脚本
    postrotate
        /bin/kill -USR1 `cat /usr/local/nginx/logs/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
' > /etc/logrotate.d/nginx
```

```bash
crontab -e
# 每天 23点59分进行日志切割
59 23 * * * /usr/sbin/logrotate -f /etc/logrotate.d/nginx
```

## 2. 模块

### 2.1 http健康检查


下载地址：https://github.com/yaoweibin/nginx_upstream_check_module


```bash
yum -y install patch
mv /opt/software/nginx_upstream_check_module /opt/software/nginx-1.22.1/modules/nginx_upstream_check_module
cd /opt/software/nginx-1.22.1
# 打补丁
patch -p1 < /opt/software/nginx-1.22.1/modules/nginx_upstream_check_module/check_1.20.1+.patch
# 编译
./configure --prefix=/usr/local/nginx --pid-path=/usr/local/nginx/logs/nginx.pid \
--user=nginx --group=nginx --build=web_server --with-pcre --with-compat --with-file-aio \
--with-stream --with-stream_ssl_module \
--with-http_ssl_module --with-http_realip_module \
--with-http_sub_module \
--with-mail_ssl_module \
--with-http_stub_status_module --with-http_gzip_static_module \
--add-module=./modules/nginx_upstream_check_module \
# 安装
make && make install
```

```nginx
location /nstatus {
    #状态页配置
    check_status;
    access_log off;
    allow   172.17.17.165;
    #允许可以连接的远端ip地址
    deny    all;
    #限制所有远端ip的连接
}
```

### 2.2 http代理

下载地址：https://github.com/chobits/ngx_http_proxy_connect_module

```bash
mv /opt/software/ngx_http_proxy_connect_module /opt/software/nginx-1.22.1/modules/ngx_http_proxy_connect_module
cd /opt/software/nginx-1.22.1
# 打补丁
patch -p1 < /opt/software/nginx-1.22.1/modules/ngx_http_proxy_connect_module/patch/proxy_connect_rewrite_102101.patch
# 编译
./configure --prefix=/usr/local/nginx --pid-path=/usr/local/nginx/logs/nginx.pid \
--user=nginx --group=nginx --build=web_server --with-pcre --with-compat --with-file-aio \
--with-stream --with-stream_ssl_module \
--with-http_ssl_module --with-http_realip_module \
--with-http_sub_module \
--with-mail_ssl_module \
--with-http_stub_status_module --with-http_gzip_static_module \
--add-module=./modules/ngx_http_proxy_connect_module \
# 安装
make && make install
```

正向代理在http模块内

```nginx
server {
    listen 3182;
    resolver 114.114.114.114;
    proxy_connect;
    proxy_connect_allow            443 80;
    proxy_connect_connect_timeout  10s;
    proxy_connect_read_timeout     10s;
    proxy_connect_send_timeout     10s;
    access_log logs/access_proxy_$logdate.log access;
    location / {
        proxy_pass $scheme://$host$request_uri;
    }
}
```

配置代理服务器地址
```bash
vi /etc/profile
export http_proxy=192.168.3.114:3182    # 正向代理
export https_proxy=192.168.3.114:3182   # 代理服务
source /etc/profile
```

win客户端验证

![](../../assets/_images/deploy/nginx/image1.png)

![](../../assets/_images/deploy/nginx/image2.png)

![](../../assets/_images/deploy/nginx/image3.png)

linux客户端验证
```bash
curl -i https://openapi.alipay.com/gateway.do
#如果未加环境变量代理设置，则可以通过临时代理访问
curl -i --proxy 192.168.3.114:3182  https://openapi.alipay.com/gateway.do
```

### 2.3 tcp代理

下载地址：https://github.com/yaoweibin/nginx_tcp_proxy_module

```bash
mv /opt/software/nginx_tcp_proxy_module /opt/software/nginx-1.22.1/modules/nginx_tcp_proxy_module
cd /opt/software/nginx-1.22.1
# 打补丁
patch -p1 < /opt/software/nginx-1.22.1/modules/nginx_tcp_proxy_module/tcp.patch
# 编译
./configure --prefix=/usr/local/nginx --pid-path=/usr/local/nginx/logs/nginx.pid \
--user=nginx --group=nginx --build=web_server --with-pcre --with-compat --with-file-aio \
--with-stream --with-stream_ssl_module \
--with-http_ssl_module --with-http_realip_module \
--with-http_sub_module \
--with-mail_ssl_module \
--with-http_stub_status_module --with-http_gzip_static_module \
--add-module=./modules/nginx_tcp_proxy_module
# 安装
make && make install
```

Nginx高于1.20.1版本编译报错，插件源码在`ngx_tcp_ssl_module.c`文件中搜索`ngx_ssl_rsa512_key_callback`注释掉可以解决编译报错问题

### 2.4 目录索引

下载地址：https://codeload.github.com/aperezdc/ngx-fancyindex/zip/master

#### 2.4.1 解压编译

```bash
cd /opt/software
unzip ngx-fancyindex-master.zip
mv /opt/software/ngx-fancyindex-master /opt/software/nginx-1.22.1/modules/ngx-fancyindex
# 编译
cd nginx-1.22.1
./configure --prefix=/usr/local/nginx --pid-path=/usr/local/nginx/logs/nginx.pid \
--user=nginx --group=nginx --build=web_server --with-pcre --with-compat --with-file-aio \
--with-stream --with-stream_ssl_module \
--with-http_ssl_module --with-http_realip_module \
--with-http_sub_module \
--with-mail_ssl_module \
--with-http_stub_status_module --with-http_gzip_static_module \
--add-module=./modules/ngx-fancyindex-master \
# 安装
make && make install
```

#### 2.4.2 基础配置

```nginx
server {
    listen 80;
    server_name www.xuzhihao.net;
    charset utf-8; 
    location / {
        root /home/www/; 
        fancyindex on;              # 开启nginx目录浏览功能
        fancyindex_exact_size off;  # 不显示精确大小
        fancyindex_localtime on;    # 显示文件修改时间为服务器本地时间
    }
}
```

#### 2.4.3 样式主题

- https://github.com/TheInsomniac/Nginx-Fancyindex-Theme
- https://github.com/Naereen/Nginx-Fancyindex-Theme
- https://github.com/fraoustin/Nginx-Fancyindex-Theme
- https://github.com/alehaa/nginx-fancyindex-flat-theme

```nginx
server {
    listen 80;
    server_name www.xuzhihao.net;
    charset utf-8; 
    location / {
        root /home/www/; 
        fancyindex on;
        fancyindex_localtime on;
        fancyindex_exact_size off;
        fancyindex_header "/fancyindex/header.html";
        fancyindex_footer "/fancyindex/footer.html";
        fancyindex_ignore "fancyindex";     # 忽略这个目录不在列表中显示
    }
}
```

> 注意：/home/www是默认站点根目录，fancyindex文件夹在`/home/www`下

#### 2.4.4 md预览

```nginx
server {
    listen   80;
    listen   [::]:80 ipv6only=on;
    listen   443 ssl;
    listen   [::]:443 default ipv6only=on ssl;
    # Enable SSL
    ssl_certificate /usr/local/nginx/cert/server.crt;
    ssl_certificate_key /usr/local/nginx/cert/private.key;
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
    location = /favicon.ico { 
        access_log off; 
        log_not_found off; 
    }
    location = /robots.txt  { 
        access_log off; 
        log_not_found off; 
    }
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

启动预览服务

```bash
/usr/local/bin/markdown-renderer -mode local -root /home/www/public/
```

> 注意：/home/www/public是默认站点根目录，应用public和fancyindex文件夹是平行关系

## 3. 高级

### 3.1 配置文件

#### 3.1.1 nginx.conf

```nginx
user nginx;
worker_processes 32;
error_log logs/error.log error;
pid logs/nginx.pid;
worker_rlimit_nofile 102400;
events {
    use epoll;
    worker_connections 102400;
}

http {
    include mime.types;
    default_type application/octet-stream;
    log_format access '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    map $time_iso8601 $logdate {
        '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
        default    'date-not-found';
    }
    access_log logs/ip_access_$logdate.log access;

    # 其他参数
    server_tokens   off;                    # 关闭错误页面中版本号
    sendfile on;                            # 优化磁盘IO设置，下载等磁盘IO高的应用，可设为off
    tcp_nopush on;                          # 提升网络传输效率，在一个数据包里发送所有头文件
    tcp_nodelay on;                         # 禁用Nagle算法，取消合并，即数据包立即发送出去
    server_names_hash_bucket_size 128;      # 服务器名字的哈希表大小
    keepalive_timeout 60s;                  # 长连接超时时间，设置为0，则表示禁用长连接
    send_timeout 30s;                       # 设置发送响应到客户端的超时时间，默认为60s
    # 请求头处理
    client_header_buffer_size 32k;          # 请求头缓冲区大小
    large_client_header_buffers 4 32k;      # 请求头最大缓冲区空间
    client_header_timeout 60s;              # 设置读取客户端请求头超时时间，默认为60s,响应408
    # 请求体处理
    client_max_body_size 10m;               # 请求主体最大限制，上传文件大小，默认值为1m（1024kb）
    client_body_buffer_size 10m;            # 请求主体缓冲区大小，如果主体大于缓冲区并且小于主体最大限制，则将文件存储到临时文件中（client_body_temp）
    client_body_timeout 60s;                # 设置读取客户端主体内容体超时时间，默认为60s，响应408
    client_body_temp_path /data/temp/;      # 请求主体临时保存路径
    client_body_in_file_only off;           # 将请求体存储在临时文件中，默认关闭。clean：请求后删除，on：请求后不删除
    client_body_in_single_buffer off;       # 默认关闭，开启优化读取$request_body变量时涉及的I/O操作
    # 代理区处理-响应缓冲区
    proxy_buffering on;                     # 开启响应缓冲区
    proxy_buffers 8 8k;                     # 代理缓冲区空间，每个worker进程可以使用8个大小为8k的缓冲区
    proxy_buffer_size 8k;                   # 单个代理缓冲区空间
    proxy_busy_buffers_size 8k;             # 设置标注“client-ready”缓冲区的最大尺寸
    proxy_max_temp_file_size 1024k;         # 当proxy_buffers放不下后端服务器的响应内容时，写入临时文件
    proxy_temp_file_write_size 64k;         # 当缓存被代理的服务器响应到临时文件时文件写入速度
    proxy_headers_hash_max_size 51200;      # 存放http报文头的哈希表容量上限，默认为512个字符。
    proxy_headers_hash_bucket_size 6400;    # nginx服务器申请存放http报文头的哈希表容量大小。默认为64个字符。
    # 代理区处理-超时时间
    proxy_connect_timeout 60s;              # 与后端/上游服务器建立连接的超时时间，默认为60s
    proxy_read_timeout 60s;                 # 设置从后端/上游服务器读取响应的超时时间，默认为60s
    proxy_send_timeout 60s;                 # 设置往后端/上游服务器发送请求的超时时间，默认为60s
    # 采用gzip压缩的形式发送数据，减少发送数据量，但会增加请求处理时间及CPU处理时间
    gzip on;
    gzip_types text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml text/javascript application/javascript application/x-javascript text/x-json application/json application/x-web-app-manifest+json text/css text/plain text/x-component font/opentype application/x-font-ttf application/vnd.ms-fontobject image/x-icon;         # 压缩文件类型
    gzip_vary on;                           # 根据请求头来判断是否需要压缩
    gzip_buffers 16 8k;                     # 缓存空间大小
    gzip_disable "MSIE [1-6]\.";            # 为指定的客户端禁用gzip功能
    gzip_http_version 1.1;                  # 使用Gzip的HTTP最低版本
    gzip_min_length 1k;                     # 小于1k字节则不压缩
    gzip_comp_level 3;                      # 压缩等级，1-9，9最慢压缩比最大
    # 文件描述符缓存
    open_file_cache max=10000 inactive=30s;    # 开启缓存的同时也指定了缓存文件的最大数量，30s如果文件没有请求则删除缓存
    open_file_cache_valid 60s;              # 60s检查一次缓存的有效信息
    open_file_cache_min_uses 5;             # 件缓存最小的访问次数，只有访问超过5次的才会被缓存
    
    include /usr/local/nginx/conf/front.conf;
    include /usr/local/nginx/conf/front_upstream.conf;
}

stream {
    log_format stream '$remote_addr [$time_local] '
                 '$protocol $status $bytes_sent $bytes_received '
                 '$session_time -> $upstream_addr '
                 '$upstream_bytes_sent $upstream_bytes_received $upstream_connect_time';
    access_log logs/stream_access_$logdate.log stream;
    include /usr/local/nginx/conf/stream_openfire.conf;
}

tcp {
    access_log logs/tcp_access_$logdate.log;
    include /usr/local/nginx/conf/tcp_openfire.conf;
}
```

#### 3.1.2 front.conf

```nginx
server {
    listen 80;
    server_name www.xuzhihao.net;
    rewrite ^(.*)$ https://$host$1 permanent;
}
server {
    listen 443 ssl;
    server_name www.xuzhihao.net;
    charset utf-8;
    ssl_certificate /usr/local/nginx/cert/server.crt;
    ssl_certificate_key /usr/local/nginx/cert/private.key;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 5m;
    ssl_buffer_size 256k;
    ssl_session_tickets on;
    # 在线查询证书吊销状态，每个机构是否验证信任链要求不一，没有则不添加，开启后如产生DNS污染问题通过resolver添加IP解决
    # ssl_stapling on;
    # ssl_stapling_verify on;
    # ssl_trusted_certificate /usr/local/nginx/cert/fullchain.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM;
    ssl_prefer_server_ciphers on;
    resolver 223.5.5.5 114.114.114.114 180.76.76.76 valid=300s;
    resolver_timeout 10s;
    index index.html index.htm index.jsp index.do index.action;
    access_log logs/www.xuzhihao.net/access_$logdate.log;
    location / {
        proxy_pass http://front;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
        add_header Access-Control-Allow-Credentials 'true';
        add_header Access-Control-Allow-Headers 'Authorization,Content-Type,X-Requested-With';
    }
}
```

#### 3.1.3 front_upstream.conf

```nginx
upstream front {
    server 172.17.17.165:5501 weight=1 max_fails=3 fail_timeout=3s;
    server 172.17.17.165:5502 weight=1 max_fails=3 fail_timeout=3s;
    check interval=2000 rise=3 fall=2 timeout=1000 type=http;
    #对front这个负载均衡池中的所有节点，每个2秒检测一次，
    #请求3次正常则标记realserver状态为up，如果检测2次都失败，则标记realserver的状态为down，后端健康请求的超时时间为1s，健
    #康检查包的类型为http请求。
    check_http_send "HEAD / HTTP/1.0\r\n\r\n";
    #通过http HEAD消息头检测realserver的健康
    check_http_expect_alive http_2xx http_3xx;
    #该指令指定HTTP回复的成功状态，默认为2XX和3XX的状态是健康的
}
```

#### 3.1.4 tcp_openfire.conf

```nginx
timeout 60000;
proxy_read_timeout 60000;
proxy_send_timeout 60000;
proxy_connect_timeout 60000;
upstream op5222{
        ip_hash;
        server 172.17.16.51:5222;
        server 172.17.16.52:5222;
        server 172.17.16.53:5222;
        check interval=3000 rise=2 fall=5 timeout=1000;
}
upstream op7070{
        ip_hash;
        server 172.17.16.51:7070;
        server 172.17.16.52:7070;
        server 172.17.16.53:7070;
        check interval=3000 rise=2 fall=5 timeout=1000;
}
server{
        listen 5223 ssl;
        access_log logs/tcp/5223_$logdate.log;
        ssl_certificate      /usr/local/nginx/cert/server.pem;
        ssl_certificate_key  /usr/local/nginx/cert/private.key;
        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;
        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;
        proxy_connect_timeout 15s;
        proxy_pass op5222;
        so_keepalive on;
        tcp_nodelay on;
}
server{
        listen 7443 ssl;
        access_log logs/tcp/7443_$logdate.log;
        ssl_certificate      /usr/local/nginx/cert/server.pem;
        ssl_certificate_key  /usr/local/nginx/cert/private.key;
        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;
        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;
        proxy_pass op7070;
        so_keepalive on;
        tcp_nodelay on;
}
```

#### 3.1.5 stream_openfire.conf

```nginx
upstream mysql {
    server 172.17.17.137:3306;
}
server {
    listen 33060;
    access_log logs/stream/33060_$logdate.log stream;
    proxy_connect_timeout 5s;
    proxy_timeout 300s;
    proxy_pass mysql;
}

upstream openfire {
    server 172.17.17.162:5222;
}
server {
    listen 25223 ssl;
    ssl_certificate      /usr/local/nginx/cert/server.crt;
    ssl_certificate_key  /usr/local/nginx/cert/private.key;
    access_log logs/stream/25223_$logdate.log stream;
    proxy_connect_timeout 5s;
    proxy_timeout 300s;
    proxy_pass openfire;
}
```

#### 3.1.5 websocket.conf

```conf
server {
    listen 80;
    server_name  www.xuzhihao.net;
    charset utf-8; 
    index index.html;
    location /ws/ {
        proxy_pass http://127.0.0.1:5032/;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 3.2 泛域名

```nginx
server {
    listen 80;
    server_name  ~^(?<serno>.+).xuzhihao.net$;
    server_name_in_redirect off; 
    location / {
        rewrite ^(.*)$ /$serno$1 break;
        root /home/www/;
        #proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    access_log logs/access_$logdate.log;
}
```


### 3.3 二级路径

```nginx
server {
    listen 80;
    server_name  www.xuzhihao.net;
    charset utf-8; 
    index index.html;
    location / {
        root /home/project;
    }
    location /download/ {               
        root  /home/project/;           # root表示物理路径，在/home/project/下必须有download文件夹，把/home/project/download作为根路径
    }
    location /emoji/ {                  
        alias /home/project/;           # alias表示虚拟路径，不对应任何文件夹，把/home/project/作为根路径
    }
    location /MP_verify_AHlGORozI4xN5yov.txt {      # wx授权域名绑定
        alias /home/MP_verify_AHlGORozI4xN5yov.txt;
    }
    location /blob/ {
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8000/blob/;
    }
}
```

> `location` 末尾有斜杠/表示转发到proxy_pass，无斜杠/表示拼接到proxy_pass，`proxy_pass` 末尾有斜杠/不拼接location的路径，无斜杠/会拼接location的路径

### 3.4 错误页

1. 指定具体跳转地址

```nginx
server {
    error_page 404 http://www.xuzhihao.net;
}
```

2. 指定重定向地址

```nginx
server {
    error_page 404 /50x.html;
    error_page 500 502 503 504 /50x.html;
    location =/50x.html {
        root html;
    }
}
```

3. 使用location的@符号

```nginx
server {
    error_page 404 @jump_to_error;
    location @jump_to_error {
        default_type text/plain;
        return 404 'Not Found Page...';
    }
}
```

4. 修改指定状态码

```nginx
server {
    error_page 404 =200 /50x.html;
    location =/50x.html {
        root html;
    }
}
```

### 3.5 跨域

```nginx
location /getUser {
    default_type application/json;
    add_header Content-Type 'text/html; charset=utf-8';
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods GET,POST,PUT,DELETE;
    if ( $query_string ~* ^(.*)v=1.0$ ) {
        return 200 '{"id":1,"name":"我是个大盗贼","age":29}';
    }
    if ( $query_string ~* ^(.*)v=2.0$ ) {
        return 200 '{"id":1,"name":"我们都是好孩子","age":29}';
    }
    if ( $request_method !~* POST ) {
        return 403;
    }
    return 200 $time_local;
}
```

### 3.6 防盗链

```nginx
location ~*\.(png|jpg|gif) {
    valid_referers none blocked www.xuzhihao.net 192.168.3.200 *.xuzhihao.net hwcq.*  ~\.hwcq\.;
    if ($invalid_referer) {
        return 403;
    }
    root /usr/local/nginx/html;
}
```

```nginx
location /images {
    valid_referers none blocked www.xuzhihao.net 192.168.3.200 *.xuzhihao.net hwcq.*  ~\.hwcq\.;
    if ($invalid_referer) {
        rewrite ^/ http://www.web.com/images/forbidden.png;
    }
    root /usr/local/nginx/html;
}
```

### 3.7 Rewrite

Rewrite作用就是使用nginx提供的全局变量或自己设置的变量，结合正则表达式和标志位实现url重写以及重定向

语法格式

```
rewrite regex replacement [flag];
```

flag标记
- last 本条规则匹配完成后， 继续向下匹配新其他uri规则
- break 本条规则匹配完成即终止， 不再匹配后面的任何规则
- redirect 返回 302 临时重定向， 浏览器地址栏会显示跳转后的url地址
- permanent 返回 301 永久重定向， 浏览器地址栏会显示跳转后的url地址

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


使用示例

```nginx

## 目录合并
server {
    listen 80;
    server_name www.web.name;
    location /server {
        rewrite ^/server-([0-9]+)-([0-9]+)-([0-9]+)-([0-9]+)\.html$ /server/$1/$2/$3/$4/$5.html last;
    }
}

## 永久重定向
server {
    listen 80;
    server_name  www.xuzhihao.net;
    location /console/ {
        rewrite ^/(.*)$ http://www.xuzhihao.net/$1 permanent;
    }
}
```


### 3.8 web缓存

```nginx
http {
    proxy_cache_path /usr/local/proxy_cache levels=2:1 keys_zone=xzh:200m inactive=1d max_size=20g; # 缓存文件的存放路径
    upstream backend {
        server 192.168.3.200:8080;
    }
    server {
        listen 8080;
        server_name  localhost;
        location / {
            if ($request_uri ~ /.*\.js$) {
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

```bash
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

```nginx
server {
    location ~ /purge(/.*) {
        proxy_cache_purge xzh xuzhihao;
    }
}
```



### 3.9 访问控制

```bash
vi blockip.conf
```

```nginx
allow 192.168.1.0/24;
allow 10.1.1.0/16;
allow 2001:0db8::/32;
deny  all;
```

```nginx
http {
    # 所有站点屏蔽规则
    # include blockip.conf;
    server {
        listen 28101;
        server_name  _;
        # 针对具体网站屏蔽规则
        # include blockip.conf;
        location / {
            proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:18101;
            # 按路径屏蔽
            # include blockip.conf;
        }
    }
}
```

### 3.10 日志分析

```bash
# 过滤指定时间内的数据输出到文件
sed -n '/31\/Jan\/2024:09/,/31\/Jan\/2024:10/p' access.log > new.log

# 统计ip或者url访问最频繁的次数
awk '{print $1}' access.log | sort | uniq -c | sort -n
# 过滤指定关键词出现次数
cat access.log | grep 'keywords' | awk '{print $1}'  | sort | uniq -c | sort -nr -k1 | head -n 10
```

### 3.11 URL美化

解决VUE项目下`#`号参数被拦截的问题

```nginx
location / {
	root /home/www/software/;
	index index.html index.htm;
	try_files $uri $uri/ /index.html;
}

location /console {
	alias /home/www/lowcode_console;
	index index.html index.htm;
	try_files $uri $uri/ /console/index.html;
}
```

## 4. 证书

### 4.1 openssl自签名

```bash
mkdir /root/cert
cd /root/cert
openssl genrsa -des3 -out private.key 2048              # 生成私钥文件
openssl req -new -key private.key -out server.csr       # 生成证书请求文件（CSR）
cp private.key private.key.org        
openssl rsa -in private.key.org -out private.key        # 利用私钥生成一个不需密码的密钥
openssl x509 -req -days 365 -in server.csr -signkey private.key -out server.crt  # 生成自签名证书365天

openssl x509 -in server.crt -noout -dates               # 验证到期时间
```

```nginx
server {
    listen       80; # 同时支持HTTP
    listen       443 ssl; # 添加HTTPS支持
    server_name  www.xuzhihao.net; #修改域名

    #ssl配置
    ssl_certificate /usr/share/nginx/html/ssl/api/server.crt;           # 配置证书
    ssl_certificate_key /usr/share/nginx/html/ssl/api/private.key;      # 配置证书私钥
    ssl_prefer_server_ciphers on;
    ssl_session_timeout 10m;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.1 TLSv1.2;
    ssl_ciphers EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256::!MD5;  # 最好的安全性，依赖openssl版本
    location / {
        proxy_pass   http://192.168.3.101:8080;         # 设置代理服务访问地址
        proxy_set_header  Host $http_host;              # 设置客户端真实的域名（包括端口号）
        proxy_set_header  X-Real-IP  $remote_addr;      # 设置客户端真实IP
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for; # 设置在多层代理时会包含真实客户端及中间每个代理服务器的IP
        proxy_set_header  X-Forwarded-Proto $scheme;    # 设置客户端真实的协议（http还是https）
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

### 4.2 Let’s Encrypt自动申请

`acme.sh`脚本实现了acme协议, 可以从`Let’s Encrypt`生成免费的证书。一般证书有效期是1年，过期需要重新申请，使用脚本可以实现到期自动申请，再也不用担心证书过期了

- 官网：https://github.com/acmesh-official/acme.sh

#### 4.2.1 安装

```bash
curl https://get.acme.sh | sh -s email=xcg992224@163.com
~/.acme.sh/acme.sh --version
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt
```

#### 4.2.2 HTTP生成

acme.sh 会自动生成验证文件，并放到站点的根目录，然后自动完成验证，最后自动删除文件

```bash
~/.acme.sh/acme.sh --issue -d test.xuzhihao.net --webroot /usr/local/nginx/html --insecure
```

 
#### 4.2.3 DNS生成

acme.sh 会自动在域名上添加一条txt解析记录, 验证域名所有权。在签发证书前需要准备阿里云账户的Accesskey，打开以下地址并创建AccessKey即可：https://usercenter.console.aliyun.com/#/manage/ak

```bash
export Ali_Key="LTAI5tAGS2KAXbgF1yWib1U3"
export Ali_Secret="xxxxxxxxxx"
source ~/.bashrc
~/.acme.sh/acme.sh --issue --dns dns_ali -d xuzhihao.net -d test.xuzhihao.net   # 单证书
~/.acme.sh/acme.sh --issue --dns dns_ali -d *.xuzhihao.net                    # 泛证书
```

#### 4.2.4 配置证书

```nginx
ssl_certificate /etc/nginx/cert.d/fullchain.cer;
ssl_certificate_key /etc/nginx/cert.d/xuzhihao.net.key;
```

相关文件的用途如下:
- ca.cer：Let’s Encrypt的中级证书
- fullchain.cer：包含中级证书的域名证书
- test.xuzhihao.net.cer：无中级证书的域名证书
- test.xuzhihao.net.conf：该域名的配置文件
- test.xuzhihao.net.csr：该域名的CSR证书请求文件
- test.xuzhihao.net.csr.conf：该域名的CSR请求文件的配置文件
- test.xuzhihao.net.key：该域名证书的私钥
