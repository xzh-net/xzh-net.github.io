# Turnserver 4.5.0.8

Turnserver是VoIP媒体流量NAT穿越服务器和网关。它也可以用作通用网络流量Turn服务器和网关。在使用WebRTC进行即时通讯时，需要使浏览器进行P2P通讯，但是由于NAT环境的复杂性，并不是所有情况下都能进行P2P，这时需要Turnserver来帮助客户端之间转发数据

libevent是一个事件通知库，适用于windows、linux、bsd等多种平台，内部使用select、epoll、kqueue、IOCP等系统调用管理事件机制。著名分布式缓存软件memcached也是基于libevent，coturn的底层网络部分依赖libevent

下载地址：https://github.com/coturn/coturn/wiki/Downloads

## 1. 安装

### 1.1 安装libevent

1. 下载编译

```bash
cd /opt/software
wget https://github.com/downloads/libevent/libevent/libevent-2.0.21-stable.tar.gz
tar xvfz libevent-2.0.21-stable.tar.gz -C /opt/software
cd /opt/software/libevent-2.0.21-stable
./configure
make
sudo make install
```

2. 安装验证

```bash
whereis libevent
```

运行demo

```bash
# 服务器端
/opt/libevent-2.0.21-stable/sample
./hello-world

# 客户端
nc 127.0.0.1 9995
```

### 1.2 安装依赖

```bash
yum install -y make gcc cc gcc-c++ wget openssl-devel libevent libevent-devel mysql-devel
```

### 1.3 编译安装

```bash
# 下载解压
cd /opt/software
wget https://coturn.net/turnserver/v4.5.0.8/turnserver-4.5.0.8.tar.gz
tar -zxvf turnserver-4.5.0.8.tar.gz -C /opt/software

# 编译安装
cd /opt/software/turnserver-4.5.0.8/
./configure # 编译，默认目录在/usr/local/bin/
make && make install
```

### 1.4 修改配置

```bash
cp /opt/software/turnserver-4.5.0.8/examples/etc/turnserver.conf /etc/turnserver.conf 

# 修改配置文件
vi /etc/turnserver.conf 
//***********************************************************************//
listening-port=3478         # turnserver监听UDP/TCP端口,默认为3478
tls-listening-port=5349     # turnserver监听TLS/DTLS端口,默认为5349
listening-ip=172.20.92.126  # 中继服务器的监听IP地址,可以配置多个。内部ip，阿里云服务器注释此行
relay-ip=172.20.92.126      # 中继服务器的IP地址
external-ip=39.105.58.136   # 外部IP,当中继服务器在NAT网络内部时指定,此处可以不添加
relay-threads=50
lt-cred-mech
user=xzh:123456
realm=xuzhihao.net
//**************************************************************************************//
```

### 1.5 添加用户

```bash
# 添加用户 -r <realm> 为该用户所属的 Realm。在启动 turnserver 时需要指定 Realm ，只有该 Realm 下的用户才能登录
turnadmin -a -u xzh -p 123456 -r xuzhihao.net
```

### 1.6 生成证书

```bash
openssl req -x509 -newkey rsa:2048 -keyout /etc/turn_server_pkey.pem -out /etc/turn_server_cert.pem -days 99999 -nodes
```

### 1.7 启动服务

```bash
turnserver -v -r 39.105.58.136:3478 -a -o -c /etc/turnserver.conf
which turnserver    # 查看安装路径
turnserver --help   # 参数说明
```

## 2. 客户端测试

```bash
turnutils_uclient -u xzh  -w 123456  -p PORT -v 39.105.58.136
```

检测网站：https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/

![](../../assets/_images/deploy/turnserver/trickle-ice.png)