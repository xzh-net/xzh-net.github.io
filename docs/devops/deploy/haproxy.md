# HAProxy 2.3.10


负载均衡策略: 

| 策略               | 含义                                                         |
| ------------------ | ------------------------------------------------------------ |
| roundrobin         | 表示简单的轮循，即客户端每访问一次，请求轮循跳转到后端不同的节点机器上 |
| static-rr          | 基于权重轮循，根据权重轮循调度到后端不同节点                 |
| leastconn          | 加权最少连接，表示最少连接者优先处理                         |
| source             | 表示根据请求源IP，这个跟Nginx的IP_hash机制类似，使用其作为解决session问题的一种方法 |
| uri                | 表示根据请求的URL，调度到后端不同的服务器                    |
| url_param          | 表示根据请求的URL参数来进行调度                              |
| hdr（name）        | 表示根据HTTP请求头来锁定每一次HTTP请求                       |
| rdp-cookie（name） | 表示根据cookie（name）来锁定并哈希每一次TCP请求              |

## 1. 下载

各版本连接：https://src.fedoraproject.org/repo/pkgs/haproxy/
```bash
cd ~
curl https://www.lua.org/ftp/lua-5.4.3.tar.gz > lua-5.4.3.tar.gz 
curl https://www.haproxy.org/download/2.3/src/haproxy-2.3.10.tar.gz > haproxy-2.3.10.tar.gz
```

## 2. 安装依赖
```bash
yum -y install make gcc gcc-c++ pcre-devel bzip2-devel openssl-devel systemd-devel
```

## 3. 添加用户
```bash
useradd -r -M -s /sbin/nologin haproxy
```

## 4. 安装lua

```bash
tar zxf lua-5.4.3.tar.gz 
cd lua-5.4.3 
make INSTALL_TOP=/usr/local/lua-5.4.3 linux install
```

## 5. 安装openssl 1.1.1k

生成动态链接库

```bash
curl -R -O https://www.openssl.org/source/openssl-1.1.1k.tar.gz 
tar xvzf openssl-1.1.1k.tar.gz 
cd openssl-1.1.1k 
./config --prefix=/usr/local/openssl
make && make install
ln -sf /usr/local/openssl/bin/openssl /usr/bin/openssl
echo "/usr/local/lib64" >> /etc/ld.so.conf
ldconfig -v
openssl version
```

## 6. 编译haproxy

```bash
tar xvf haproxy-2.3.10.tar.gz 
cd haproxy-2.3.10 
uname -r # 查看内核
# 如：3.10.0-1062.el7.x86_64 对应为 TARGET=linux3100 ARCH=x86_64
make ARCH=x86_64 \ TARGET=linux-glibc \ USE_NS=1 \ USE_TFO=1 \ USE_OPENSSL=1 \ SSL_LIB=/usr/local/openssl/lib \ SSL_INC=/usr/local/openssl/include \ USE_ZLIB=1 \ USE_LUA=1 \ USE_PCRE=1 \ USE_SYSTEMD=1 \ USE_LIBCRYPT=1 \ USE_THREAD=1 \ USE_CPU_AFFINITY=1 \ LUA_INC=/usr/local/lua-5.4.3/include \ LUA_LIB=/usr/local/lua-5.4.3/lib \ EXTRA_OBJS="contrib/prometheus-exporter/service-prometheus.o" \ PREFIX=/usr/local/haproxy 

make install PREFIX=/usr/local/haproxy 
cp -a haproxy /usr/sbin/
file haproxy
haproxy -v 
```

## 7. 配置负载内核

```bash
echo 'net.ipv4.ip_nonlocal_bind = 1' >>  /etc/sysctl.conf
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl  -p
```

## 8. 配置文件

```conf
  global
         log 127.0.0.1 local3        # 定义haproxy日志输出设置
         log 127.0.0.1   local1 notice        
         #log loghost    local0 info # 定义haproxy 日志级别
         ulimit-n 82000              # 设置每个进程的可用的最大文件描述符
         maxconn 20480               # 默认最大连接数
         chroot /usr/local/haproxy   # chroot运行路径
         uid 99                      # 运行haproxy 用户 UID
         gid 99                      # 运行haproxy 用户组gid
         daemon                      # 以后台形式运行harpoxy
         nbproc 1                    # 设置进程数量
         pidfile /usr/local/haproxy/run/haproxy.pid  #haproxy 进程PID文件
         #debug                      # haproxy调试级别，建议只在开启单进程的时候调试
         #quiet

defaults
         log    global           	# 引入global定义的日志格式
         mode    http          		# 所处理的类别(7层代理http，4层代理tcp)
         maxconn 50000         		# 最大连接数
         option  httplog       		# 日志类别为http日志格式
         option  httpclose     		# 每次请求完毕后主动关闭http通道
         option  dontlognull   		# 不记录健康检查日志信息
         option  forwardfor    		# 如果后端服务器需要获得客户端的真实ip，需要配置的参数，可以从http header 中获取客户端的IP
         retries 3             		# 3次连接失败就认为服务器不可用，也可以通过后面设置        
         stats refresh 30       	# 设置统计页面刷新时间间隔
         option abortonclose    	# 当服务器负载很高的时候，自动结束掉当前队列处理比较久的连接
         balance roundrobin     	# 设置默认负载均衡方式，轮询方式
         #balance source        	# 设置默认负载均衡方式，类似于nginx的ip_hash      
         #contimeout 5000        	# 设置连接超时时间
         #clitimeout 50000       	# 设置客户端超时时间
         #srvtimeout 50000       	# 设置服务器超时时间
         timeout http-request    10s  	# 默认http请求超时时间
         timeout queue           1m   	# 默认队列超时时间
         timeout connect         10s  	# 默认连接超时时间
         timeout client          1m   	# 默认客户端超时时间
         timeout server          1m   	# 默认服务器超时时间
         timeout http-keep-alive 10s  	# 默认持久连接超时时间
         timeout check           10s  	# 设置超时检查超时时间


frontend http_80_in
    	 bind 0.0.0.0:80    	  # 设置监听端口，即haproxy提供的web服务端口，和lvs的vip 类似
     	mode http          		  # http 的7层模式
     	log global          	  # 应用全局的日志设置
     	option httplog            # 启用http的log
     	option httpclose          # 每次请求完毕后主动关闭http通道，HAproxy不支持keep-alive模式     
     	option forwardfor         # 如果后端服务器需要获得客户端的真实IP需要配置此参数，将可以从HttpHeader中获得客户端IP
     	default_backend wwwpool   # 设置请求默认转发的后端服务池


backend wwwpool      					# 定义wwwpool服务器组。
  		mode http           			# http的7层模式
  		option  redispatch
  		option  abortonclose
  		balance source      			# 负载均衡的方式，源哈希算法
  		cookie  SERVERID    			# 允许插入serverid到cookie中，serverid后面可以定义
  		option  httpchk GET /test.html  # 心跳检测
  		server web1 10.1.1.2:80 cookie 2 weight 3 check inter 2000 rise 2 fall 3 maxconn 8

listen  admin_status           			# Frontend和Backend的组合体,监控组的名称，按需自定义名称 
         bind 0.0.0.0:8888              # 监听端口 
         mode http                      # http的7层模式 
         log 127.0.0.1 local3 err       # 错误日志记录 
         stats refresh 5s               # 每隔5秒自动刷新监控页面 
         stats uri /admin?stats         # 监控页面的url访问路径 
         stats realm itnihao\ welcome   # 监控页面的提示信息 
         stats auth admin:admin         # 监控页面的用户和密码admin,可以设置多个用户名 
         stats auth admin1:admin1       # 监控页面的用户和密码admin1 
         stats hide-version             # 隐藏统计页面上的HAproxy版本信息  
         stats admin if TRUE            # 手工启用/禁用,后端服务器(haproxy-1.4.9以后版本)
```

## 9. 启动

```bash
/usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/conf/haproxy.conf
ps -ef | grep haproxy
haproxy -f /usr/local/haproxy/conf/haproxy.cfg
```

## 10. 代理

### 10.1 RabbitMQ

- 192.168.13.104	HAProxy	server4
- 192.168.13.105	HAProxy	server5

```conf
#logging options
global
    log 127.0.0.1 local0 info
    maxconn 5120
    # ha的安装地址
    chroot /usr/local/haproxy
    uid 99
    gid 99
    daemon
    quiet
    nbproc 20
    pidfile /var/run/haproxy.pid
​
defaults
    log global
    #使用4层代理模式，”mode http”为7层代理模式
    mode tcp
    #if you set mode to tcp,then you nust change tcplog into httplog
    option tcplog
    option dontlognull
    retries 3
    option redispatch
    maxconn 2000
    contimeout 5s
     ##客户端空闲超时时间为 30秒 则HA 发起重连机制
     clitimeout 30s
     ##服务器端链接超时时间为 15秒 则HA 发起重连机制
     srvtimeout 15s 
#front-end IP for consumers and producters
​
listen rabbitmq_cluster
    bind 192.168.13.104:5672
    #配置TCP模式
    mode tcp
    #balance url_param userid
    #balance url_param session_id check_post 64
    #balance hdr(User-Agent)
    #balance hdr(host)
    #balance hdr(Host) use_domain_only
    #balance rdp-cookie
    #balance leastconn
    #balance source //ip
    #简单的轮询
    balance roundrobin
    #rabbitmq集群节点配置 #inter 每隔五秒对mq集群做健康检查， 2次正确证明服务器可用，2次失败证明服务器不可用，并且配置主备机制
        server server1 192.168.13.101:5672 check inter 5000 rise 2 fall 2
        server server2 192.168.13.102:5672 check inter 5000 rise 2 fall 2
        server server3 192.168.13.103:5672 check inter 5000 rise 2 fall 2
#配置haproxy web监控，查看统计信息
listen stats
    bind 192.168.13.104:8100 # 注意此处的ip地址,我们配置了2台机器
    mode http
    option httplog
    stats enable
    #设置haproxy监控地址为http://192.168.13.104:8100/rabbitmq-stats
    stats uri /rabbitmq-stats
    stats refresh 5s
```

```conf
#logging options
global
    log 127.0.0.1 local0 info
    maxconn 5120
    # ha的安装地址
    chroot /usr/local/haproxy
    uid 99
    gid 99
    daemon
    quiet
    nbproc 20
    pidfile /var/run/haproxy.pid
​
defaults
    log global
    #使用4层代理模式，”mode http”为7层代理模式
    mode tcp
    #if you set mode to tcp,then you nust change tcplog into httplog
    option tcplog
    option dontlognull
    retries 3
    option redispatch
    maxconn 2000
    contimeout 5s
     ##客户端空闲超时时间为 30秒 则HA 发起重连机制
     clitimeout 30s
     ##服务器端链接超时时间为 15秒 则HA 发起重连机制
     srvtimeout 15s 
#front-end IP for consumers and producters
​
listen rabbitmq_cluster
    bind 192.168.13.105:5672
    #配置TCP模式
    mode tcp
    #balance url_param userid
    #balance url_param session_id check_post 64
    #balance hdr(User-Agent)
    #balance hdr(host)
    #balance hdr(Host) use_domain_only
    #balance rdp-cookie
    #balance leastconn
    #balance source //ip
    #简单的轮询
    balance roundrobin
    #rabbitmq集群节点配置 #inter 每隔五秒对mq集群做健康检查， 2次正确证明服务器可用，2次失败证明服务器不可用，并且配置主备机制
        server server1 192.168.13.101:5672 check inter 5000 rise 2 fall 2
        server server2 192.168.13.102:5672 check inter 5000 rise 2 fall 2
        server server3 192.168.13.103:5672 check inter 5000 rise 2 fall 2
#配置haproxy web监控，查看统计信息
listen stats
    bind 192.168.13.105:8100 # 注意此处的ip地址,我们配置了2台机器
    mode http
    option httplog
    stats enable
    #设置haproxy监控地址为http://192.168.13.105:8100/rabbitmq-stats
    stats uri /rabbitmq-stats
    stats refresh 5s
```

### 10.2 Mycat

vim /usr/local/haproxy/haproxy.conf

```yml
global
	log 127.0.0.1 local0 
	maxconn 4096 
	chroot /usr/local/haproxy 
	pidfile /usr/data/haproxy/haproxy.pid
	uid 99
	gid 99
	daemon
	node mysql-haproxy-01
	description mysql-haproxy-01
defaults
	log global
	mode tcp
	option abortonclose
	option redispatch
	retries 3
	maxconn 2000
	timeout connect 50000ms
	timeout client 50000ms
	timeout server 50000ms
listen proxy_status
	bind 0.0.0.0:48066
		mode tcp
		balance roundrobin
		server mycat_1 192.168.192.157:8066 check
		server mycat_2 192.168.192.158:8066 check
frontend admin_stats
	bind 0.0.0.0:8888
		mode http
		stats enable
		option httplog
		maxconn 10
		stats refresh 30s
		stats uri /admin
		stats auth admin:123123
		stats hide-version
		stats admin if TRUE
```



**内容解析如下** : 

```yml
#global 配置中的参数为进程级别的参数，通常与其运行的操作系统有关
global
	#定义全局的syslog服务器, 最多可定义2个; local0 是日志设备, 对应于/etc/rsyslog.conf中的配置 , 默认收集info级别日志
	log 127.0.0.1 local0 
	#log 127.0.0.1 local1 notice
	#log loghost local0 info
	#设定每个haproxy进程所接受的最大并发连接数 ;
	maxconn 4096 
	#修改HAproxy工作目录至指定的目录并在放弃权限之前执行chroot操作, 可以提升haproxy的安全级别
	chroot /usr/local/haproxy 
	#进程ID保存文件
	pidfile /usr/data/haproxy/haproxy.pid
	#指定用户ID
	uid 99
	#指定组ID
	gid 99
	#设置HAproxy以守护进程方式运行
	daemon
	#debug
	#quiet
	node mysql-haproxy-01  ## 定义当前节点的名称，用于 HA 场景中多 haproxy 进程共享同一个 IP 地址时
	description mysql-haproxy-01 ## 当前实例的描述信息
	
#defaults：用于为所有其他配置段提供默认参数，这默认配置参数可由下一个"defaults"所重新设定
defaults
	#继承global中的log定义
	log global
	#所使用的处理模式(tcp:四层 , http:七层, health:状态检查,只返回OK)
	### tcp: 实例运行于纯 tcp 模式，在客户端和服务器端之间将建立一个全双工的连接，且不会对 7 层报文做任何类型的检查，此为默认模式
	### http:实例运行于 http 模式，客户端请求在转发至后端服务器之前将被深度分析，所有不与 RFC 模式兼容的请求都会被拒绝
	### health：实例运行于 health 模式，其对入站请求仅响应“OK”信息并关闭连接，且不会记录任何日志信息 ，此模式将用于相应外部组件的监控状态检测请求
	mode tcp
	#当服务器负载很高的时候，自动结束掉当前队列处理时间比较长的连接
	option abortonclose
		
	#当使用了cookie时，haproxy将会将请求的后端服务器的serverID插入到cookie中，以保证会话的session持久性，而此时，后端服务器宕机，但是客户端的cookie不会刷新，设置此参数，将会将客户请求强制定向到另外一个后端server上，以保证服务的正常。
	option redispatch
	retries 3
	# 前端的最大并发连接数（默认为 2000）
	maxconn 2000
	# 连接超时(默认是毫秒,单位可以设置 us,ms,s,m,h,d)
	timeout connect 5000
	# 客户端超时时间
	timeout client 50000
	# 服务器超时时间
	timeout server 50000

#listen: 用于定义通过关联“前端”和“后端”一个完整的代理，通常只对 TCP 流量有用
listen proxy_status
	bind 0.0.0.0:48066 # 绑定端口
		mode tcp
		balance roundrobin # 定义负载均衡算法，可用于"defaults"、"listen"和"backend"中,默认为轮询
		#格式: server <name> <address> [:[port]] [param*]
		# weight : 权重，默认为 1，最大值为 256，0 表示不参与负载均衡
        # backup : 设定为备用服务器，仅在负载均衡场景中的其他 server 均不可以启用此 server
        # check  : 启动对此 server 执行监控状态检查，其可以借助于额外的其他参数完成更精细的设定
        # inter  : 设定监控状态检查的时间间隔，单位为毫秒，默认为 2000，也可以使用 fastinter 和 downinter 来根据服务器端专题优化此事件延迟
        # rise   : 设置 server 从离线状态转换至正常状态需要检查的次数（不设置的情况下，默认值为 2）
        # fall   : 设置 server 从正常状态转换至离线状态需要检查的次数（不设置的情况下，默认值为 3）
        # cookie : 为指定 server 设定 cookie 值，此处指定的值将会在请求入站时被检查，第一次为此值挑选的 server 将会被后续的请求所选中，其目的在于实现持久连接的功能
        # maxconn: 指定此服务器接受的最大并发连接数，如果发往此服务器的连接数目高于此处指定的值，其将被放置于请求队列，以等待其他连接被释放
		server mycat_1 192.168.192.157:8066 check inter 10s
		server mycat_2 192.168.192.158:8066 check inter 10s

# 用来匹配接收客户所请求的域名，uri等，并针对不同的匹配，做不同的请求处理
# HAProxy 的状态信息统计页面
frontend admin_stats
	bind 0.0.0.0:8888
		mode http
		stats enable
		option httplog
		maxconn 10
		stats refresh 30s
		stats uri /admin
		stats auth admin:123123
		stats hide-version
		stats admin if TRUE
```