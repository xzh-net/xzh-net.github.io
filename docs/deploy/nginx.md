# Nginx 1.20.2

- 官方网站：http://nginx.org

## 1. 安装

### 1.1 编译安装

#### 1.1.1 下载

```bash
cd /opt/software
wget http://nginx.org/download/nginx-1.20.2.tar.gz
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
tar zxvf nginx-1.20.2.tar.gz
cd nginx-1.20.2
./configure --prefix=/usr/local/nginx --pid-path=/usr/local/nginx/logs/nginx.pid --user=nginx --group=nginx --build=web_server --with-threads  --with-file-aio --with-http_v2_module --with-http_ssl_module --with-stream --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module
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

```bash
echo '/usr/local/nginx/logs/*.log {
    su root root
    daily
    rotate 5
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
logrotate -d /etc/logrotate.d/nginx
logrotate -f /etc/logrotate.d/nginx
```


## 2. 模块

### 2.1 http健康检查


下载地址：https://github.com/yaoweibin/nginx_upstream_check_module

https://github.com/xzh-net/other/tree/main/nginx/nginx_upstream_check_module


```bash
yum -y install patch
mv /opt/software/nginx_upstream_check_module /opt/software/nginx-1.22.1/modules/nginx_upstream_check_module
cd /opt/software/nginx-1.22.1

patch -p1 < /opt/software/nginx-1.22.1/modules/nginx_upstream_check_module/check_1.20.1+.patch
./configure --prefix=/usr/local/nginx --pid-path=/usr/local/nginx/logs/nginx.pid --user=nginx --group=nginx --build=web_server --with-threads  --with-file-aio --with-http_v2_module --with-http_ssl_module --with-stream --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --add-module=./modules/nginx_upstream_check_module
make && make install
```

```conf
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

### 2.2 正向代理

下载地址：https://github.com/chobits/ngx_http_proxy_connect_module

https://github.com/xzh-net/other/tree/main/nginx/ngx_http_proxy_connect_module

```bash
mv /opt/software/ngx_http_proxy_connect_module /opt/software/nginx-1.22.1/modules/ngx_http_proxy_connect_module
cd /opt/software/nginx-1.22.1

patch -p1 < /opt/software/nginx-1.22.1/modules/ngx_http_proxy_connect_module/patch/proxy_connect_rewrite_102101.patch
./configure --prefix=/usr/local/nginx --pid-path=/usr/local/nginx/logs/nginx.pid --user=nginx --group=nginx --build=web_server --with-threads  --with-file-aio --with-http_v2_module --with-http_ssl_module --with-stream --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --add-module=./modules/nginx_upstream_check_module --add-module=./modules/ngx_http_proxy_connect_module
make && make install
```

正向代理在http模块内

```conf
server {
    listen 3182;
    resolver 114.114.114.114;
    proxy_connect;
    proxy_connect_allow            443 80;
    proxy_connect_connect_timeout  10s;
    proxy_connect_read_timeout     10s;
    proxy_connect_send_timeout     10s;
    access_log logs/access_proxy.log main;
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

### 2.3 目录索引

#### 2.3.1 下载插件

```bash
cd /opt/software
wget https://codeload.github.com/aperezdc/ngx-fancyindex/zip/master -O ngx-fancyindex-master.zip    # 插件
```

#### 2.3.2 解压编译

```bash
unzip ngx-fancyindex-master.zip
mv ngx-fancyindex-master /opt
cd nginx-1.20.2
./configure  --prefix=/usr/local/nginx --with-http_stub_status_module  --user=nginx --group=nginx --with-http_ssl_module --add-module=../ngx-fancyindex-master
make  # 不要 make install
cp objs/nginx  /usr/local/nginx/sbin/  # 复制重新编译的nginx文件到nginx原来安装目录下
```

#### 2.3.3 配置模板


1. 下载模板

```bash
wget https://codeload.github.com/xzh-net/nginx-fancyindex/zip/refs/heads/main -O nginx-fancyindex-main.zip -P /home/www  # 模板
cd /home/www
unzip nginx-fancyindex.zip	
mv nginx-fancyindex fancyindex
```

2. 配置fancyindex路径

> 注意：/home/www是默认站点根目录，fancyindex文件夹在`/home/www`下

```conf
vi /usr/local/nginx/conf/nginx.conf

# 修改内容
server {
    listen 80;
    server_name www.xuzhihao.net;
    charset utf-8; 
    location / {
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

#### 2.3.4 md在线预览

> 注意：/home/www/public是默认站点根目录，应用public和fancyindex文件夹是平行关系

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

#### 2.3.5 启动服务

```
/usr/local/bin/markdown-renderer -mode local -root /home/www/public/
```

## 3. 高级

### 3.1 配置文件

#### 3.1.1 nginx.conf

```conf
user nginx;
worker_processes 32;
error_log logs/error.log info;
pid logs/nginx.pid;
worker_rlimit_nofile 102400;
events {
    use epoll;
    worker_connections 102400;
}

http {
    include mime.types;
    default_type application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    # 日志规则
    map $time_iso8601 $logdate {
        '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
        default    'date-not-found';
    }
    access_log logs/access_$logdate.log main;
    # 其他参数
    server_tokens   off;                    # 关闭错误页面中版本号
    sendfile on;                            # 优化磁盘IO设置，下载等磁盘IO高的应用，可设为off
    tcp_nopush on;                          # 提升网络传输效率，在一个数据包里发送所有头文件
    tcp_nodelay on;                         # 禁用Nagle算法，取消合并，即数据包立即发送出去
    server_names_hash_bucket_size 128;      # 服务器名字的哈希表大小
    keepalive_timeout 60s;                  # 长连接超时时间,设置为0，则表示禁用长连接
    send_timeout 30s;                       # 设置发送响应到客户端的超时时间，默认为60s
    # 请求头处理
    client_header_buffer_size 32k;          # 请求头缓冲区大小
    large_client_header_buffers 4 32k;      # 请求头最大缓冲区空间
    client_header_timeout 60s;              # 设置读取客户端请求头超时时间，默认为60s,响应408
    # 请求体处理
    client_max_body_size 1m;                # 请求主体最大限制，上传文件大小，默认值为1m（1024kb）
    client_body_buffer_size 16k;            # 请求主体缓冲区大小，如果主体大于缓冲区并且小于主体最大限制，则将文件存储到临时文件中（client_body_temp）
    client_body_timeout 60s;                # 设置读取客户端主体内容体超时时间，默认为60s,响应408
    client_body_temp_path /data/temp/;      # 请求主体临时保存路径
    client_body_in_file_only off;           # 将请求体存储在临时文件中，默认关闭。clean：请求后删除,on：请求后不删除
    client_body_in_single_buffer off;       # 默认关闭，开启优化读取$request_body变量时涉及的I/O操作
    # 代理区处理-响应缓冲区
    proxy_buffering on;                     # 开启响应缓冲区
    proxy_buffers 8 8k;                     # 代理缓冲区空间，每个worker进程可以使用8个大小为8k的缓冲区
    proxy_buffer_size 8k;                   # 单个代理缓冲区空间
    proxy_busy_buffers_size 8k;             # 设置标注“client-ready”缓冲区的最大尺寸
    proxy_max_temp_file_size 1024k;         # 当proxy_buffers放不下后端服务器的响应内容时,写入临时文件
    proxy_temp_file_write_size 64k;         # 当缓存被代理的服务器响应到临时文件时文件写入速度
    proxy_headers_hash_max_size 51200;      # 存放http报文头的哈希表容量上限，默认为512个字符。
    proxy_headers_hash_bucket_size 6400;    # nginx服务器申请存放http报文头的哈希表容量大小。默认为64个字符。
    # 代理区处理-超时时间
    proxy_connect_timeout 60s;              # 与后端/上游服务器建立连接的超时时间，默认为60s，此时间不超过75s
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
    log_format proxy '$remote_addr [$time_local] '
                 '$protocol $status $bytes_sent $bytes_received '
                 '$session_time -> $upstream_addr '
                 '$upstream_bytes_sent $upstream_bytes_received $upstream_connect_time';

    access_log logs/tcp_access.log proxy;
    include /usr/local/nginx/conf/ssh.conf;
    include /usr/local/nginx/conf/mysql.conf;
    include /usr/local/nginx/conf/openfire.conf;
}

include /usr/local/nginx/conf/openfire.conf;
```

#### 3.1.2 front.conf

```conf
server {
    listen 80;
    server_name www.xuzhihao.net;
    rewrite ^(.*)$ https://$host$1 permanent;
}
server {
    listen 443 ssl;
    server_name www.xuzhihao.net;
    charset utf-8;
    ssl_certificate /usr/local/nginx/cert/www.xuzhihao.net_chain.crt;
    ssl_certificate_key /usr/local/nginx/cert/www.xuzhihao.net_key.key;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 5m;
    ssl_buffer_size 256k;
    ssl_session_tickets on;
    ssl_stapling on;
    ssl_stapling_verify on;
    # 在线查询证书吊销状态，每个机构是否验证信任链要求不一，没有则不添加，开启后如产生DNS污染问题通过resolver添加IP解决
    # ssl_trusted_certificate /usr/local/nginx/cert/fullchain.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM;
    ssl_prefer_server_ciphers on;
    resolver 223.5.5.5 114.114.114.114 180.76.76.76 valid=300s;
    resolver_timeout 10s;
    index index.html index.htm index.jsp index.do index.action;
    access_log logs/www.xuzhihao.net/access_${logdate}.log;
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

```conf
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

#### 3.1.4 openfire.conf

```conf
upstream op5222 {
    server 172.17.16.51:5222 weight=5 max_fails=3 fail_timeout=30s;
    server 172.17.16.52:5222 weight=5 max_fails=3 fail_timeout=30s;
    server 172.17.16.53:5222 weight=5 max_fails=3 fail_timeout=30s;
}
upstream op7070 {
    server 172.17.16.51:7070 weight=5 max_fails=3 fail_timeout=30s;
    server 172.17.16.52:7070 weight=5 max_fails=3 fail_timeout=30s;
    server 172.17.16.53:7070 weight=5 max_fails=3 fail_timeout=30s;
}
server {
    listen 5223 ssl;
    proxy_connect_timeout 5s;
    proxy_timeout 300s;
    access_log logs/tcp.5223.log tcp;
    ssl_certificate      /usr/local/nginx/cert/vjsp.cn.pem;
    ssl_certificate_key  /usr/local/nginx/cert/vjsp.cn.key;
    ssl_session_cache    shared:SSL:10m;
    ssl_session_timeout  10m;
    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;
    proxy_pass op5222;    
}
server {
    listen 7443 ssl;
    proxy_connect_timeout 5s;
    proxy_timeout 300s;
    access_log logs/tcp.7443.log tcp;
    ssl_certificate      /usr/local/nginx/cert/vjsp.cn.pem;
    ssl_certificate_key  /usr/local/nginx/cert/vjsp.cn.key;
    ssl_session_cache    shared:SSL:10m;
    ssl_session_timeout  10m;
    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;
    proxy_pass op7070;
}
```

#### 3.1.5 mysql.conf

```conf
upstream myback {
    server 192.168.3.200:3306;
    server 192.168.3.201:3306;
}
server {
    listen 3301;
    proxy_connect_timeout 5s;
    proxy_timeout 300s;
    proxy_pass myback;
}
```

#### 3.1.6 ssh.conf

```conf
upstream back {
    server 172.17.17.161:22;
}
server {
    listen 2000;
    proxy_connect_timeout 5s;
    proxy_timeout 300s;
    proxy_pass back;
}
```


### 3.4 泛域名

```bash
./configure  --prefix=/usr/local/nginx --with-http_sub_module
```

```conf
server {
    listen 80;
    server_name  ~^(?<serno>.+).xuzhihao.net$;
    server_name_in_redirect off; 
    location / {
        rewrite ^(.*)$ /$serno$1 break;
        #root D:/workspace/;
        #proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
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

### 3.6 错误页

1. 指定具体跳转地址

```conf
server {
    error_page 404 http://www.xuzhihao.net;
}
```

2. 指定重定向地址

```conf
server {
    error_page 404 /50x.html;
    error_page 500 502 503 504 /50x.html;
    location =/50x.html {
        root html;
    }
}
```

3. 使用location的@符号

```conf
server {
    error_page 404 @jump_to_error;
    location @jump_to_error {
        default_type text/plain;
        return 404 'Not Found Page...';
    }
}
```

4. 修改指定状态码

```conf
server {
    error_page 404 =200 /50x.html;
    location =/50x.html {
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
    if ( $query_string ~* ^(.*)v=1.0$ ) {
        return 200 '{"id":1,"name":"我是个大盗贼","age":29}';
    }
    if ( $query_string ~* ^(.*)v=2.0$ ) {
        return 200 '{"id":1,"name":"我们都是好孩子","age":29}';
    }
    return 200 $time_local;
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
location ~*\.(png|jpg|gif) {
    valid_referers none blocked www.xuzhihao.net 192.168.3.200 *.xuzhihao.net hwcq.*  ~\.hwcq\.;
    if ($invalid_referer) {
        return 403;
    }
    root /usr/local/nginx/html;
}
```

```conf
location /images {
    valid_referers none blocked www.xuzhihao.net 192.168.3.200 *.xuzhihao.net hwcq.*  ~\.hwcq\.;
    if ($invalid_referer) {
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
    location /server {
        rewrite ^/server-([0-9]+)-([0-9]+)-([0-9]+)-([0-9]+)\.html$ /server/$1/$2/$3/$4/$5.html last;
    }
}
```

二级转发

```conf
server {
    listen 80;
    server_name  www.xuzhihao.net;
    location /console/ {
        rewrite ^/(.*)$ http://www.xuzhihao.net/$1 permanent;
    }
}
```

### 3.10 web缓存

```conf
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
server {
    location ~ /purge(/.*) {
        proxy_cache_purge xzh xuzhihao;
    }
}
```



### 3.12 IP过滤

1. 屏蔽ip

单独网站屏蔽IP的方法，把include blocksip.conf放到网址对应的在server{}语句块，所有网站屏蔽IP的方法，把include blocksip.conf放到http {}语句块。

```conf
include blockip.conf; 
# 添加内容
allow  180.164.67.212;
allow  59.46.186.253;
deny all;
```

```conf
http {
    # include blockip.conf;
    server {
        listen 28101;
        server_name  _;
        # include blockip.conf;
        location / {
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

```conf
location / {
    if ($http_user_agent ~* "scrapy|python|curl|java|wget|httpclient|okhttp") {
        return 503;
    }
}
```

### 3.13 URL美化

解决VUE项目下`#`号参数被拦截的问题

```conf
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

```conf
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

```conf
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
