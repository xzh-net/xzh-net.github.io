# TURN Server 4.5.0.8

TURN Server是VoIP媒体流量NAT穿越服务器和网关。它也可以用作通用网络流量TURN服务器和网关。在使用WebRTC进行即时通讯时，需要使浏览器进行P2P通讯，但是由于NAT环境的复杂性，并不是所有情况下都能进行P2P，这时需要TURN Server来帮助客户端之间转发数据

libevent是一个事件通知库，适用于windows、linux、bsd等多种平台，内部使用select、epoll、kqueue、IOCP等系统调用管理事件机制。著名分布式缓存软件memcached也是基于libevent，coturn的底层网络部分依赖libevent

## 1. 安装

```bash
# 编译安装 libevent
wget https://github.com/downloads/libevent/libevent/libevent-2.0.21-stable.tar.gz
tar xvfz libevent-2.0.21-stable.tar.gz
cd libevent-2.0.21-stable
./configure
make
sudo make install

# 安装依赖
yum install -y make gcc cc gcc-c++ wget openssl-devel libevent libevent-devel mysql-devel

# 安装最新版本的coturn，可以查看https://github.com/coturn/coturn/wiki/Downloads
wget https://coturn.net/turnserver/v4.5.0.8/turnserver-4.5.0.8.tar.gz

# 解压并进入目录
tar -zxvf turnserver-4.5.0.8.tar.gz
cd turnserver-4.5.0.8/

# 编译，默认目录在/usr/local/bin/
./configure
make && make install
```

## 2. 配置

```bash
# 查看是否安装成功
which turnserver

# 参数说明
turnserver --help

# 添加用户 -r <realm> 为该用户所属的 Realm。在启动 turnserver 时需要指定 Realm ，只有该 Realm 下的用户才能登录
turnadmin -a -u xzh -p 123456 -r xuzhihao.net

# 生成证书
openssl req -x509 -newkey rsa:2048 -keyout /etc/turn_server_pkey.pem -out /etc/turn_server_cert.pem -days 99999 -nodes

# 拷贝配置文件
cp turnserver-4.5.0.8/examples/etc/turnserver.conf /etc/turnserver.conf 

# 修改配置文件
vi /etc/turnserver.conf 
//***********************************************************************//
listening-ip=172.30.43.37
listening-port=3478
tls-listening-port=5349
external-ip=120.25.162.35
relay-threads=50
lt-cred-mech
user=xzh:123456
realm=xuzhihao.net
//**************************************************************************************//
```


## 3. 测试

```bash
# 启动
turnserver -c /etc/turnserver.conf -o -v

# TURN服务测试
turnutils_uclient -u xzh  -w 123456  -p PORT -v 120.25.162.35

# 检测网站
https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/

```
