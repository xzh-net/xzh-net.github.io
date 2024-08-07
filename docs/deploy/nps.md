# NPS 0.26.10

NPS是一款轻量级、高性能、功能强大的内网穿透代理服务器。目前支持tcp、udp流量转发，可支持任何tcp、udp上层协议（访问内网网站、本地支付接口调试、ssh访问、远程桌面，内网dns解析等等……），此外还支持内网http代理、内网socks5代理、p2p等，并带有功能强大的web管理端

原理：运行NPS服务的云服务器和运行NPS客户端的主机之间会创建一条TCP或UDP隧道，可以映射云服务器上的某个端口到客户端主机的指定端口，其他主机访问云服务器的这个端口的流量都会通过创建的这条隧道转发到映射的主机端口，实现内网穿透效果

- 下载地址：https://github.com/ehang-io/nps/releases

## 1. 服务端

### 1.1 二进制安装

![](../../assets/_images/deploy/nps/nps1.png)


```bash
mkdir /opt/nps/
cd /opt/nps/
tar -xvf linux_amd64_server.tar.gz
./nps install           # 输入安装命令
nps start               # 启动nps服务
```

### 1.2 编译安装

```bash
wget https://codeload.github.com/ehang-io/nps/zip/refs/heads/master -O nps-master.zip -P /home/xzh/go
cd /home/xzh/go
unzip nps-master.zip
cd nps-master
go build cmd/nps/nps.go    # 服务端
go build cmd/npc/npc.go    # 客户端
```

### 1.3 配置文件

- /etc/nps/conf/nps.conf

```conf
##bridge
bridge_type=tcp
bridge_port=8024
bridge_ip=0.0.0.0

#web
web_host=39.105.58.136      # 外网ip
web_username=admin          # 管理员账号密码
web_password=123
web_port = 8080             # 管理台端口
```

名称 | 含义
---|---
web_port | web管理端口
web_password | web界面管理密码
web_username | web界面管理账号
web_base_url | web管理主路径,用于将web管理置于代理子路径后面
bridge_port  | 服务端客户端通信端口
https_proxy_port | 域名代理https代理监听端口
http_proxy_port | 域名代理http代理监听端口
auth_key|web api密钥
bridge_type|客户端与服务端连接方式kcp或tcp
public_vkey|客户端以配置文件模式启动时的密钥，设置为空表示关闭客户端配置文件连接模式
ip_limit|是否限制ip访问，true或false或忽略
flow_store_interval|服务端流量数据持久化间隔，单位分钟，忽略表示不持久化
log_level|日志输出级别
auth_crypt_key | 获取服务端authKey时的aes加密密钥，16位
p2p_ip| 服务端Ip，使用p2p模式必填
p2p_port|p2p模式开启的udp端口
pprof_ip|debug pprof 服务端ip
pprof_port|debug pprof 端口
disconnect_timeout|客户端连接超时，单位 5s，默认值 60，即 300s = 5mins


### 1.4 WebUI

访问地址：http://39.105.58.136:8080/

![](../../assets/_images/deploy/nps/nps2.png)

![](../../assets/_images/deploy/nps/nps3.png)

## 2. 客户端

### 2.1 域名代理

适用范围： 小程序开发、微信公众号开发、产品演示

假设场景：
- 有一个`已备案`域名hwcq.online，有一台公网机器ip为39.105.58.136
- 内网开发站点127.0.0.1:8088
- 想通过www.hwcq.online访问127.0.0.1:8088

使用步骤：
- 将*.hwcq.online解析到公网服务器39.105.58.136
- 点击刚才创建的客户端的域名管理，添加规则（域名：www.hwcq.online 内网目标：127.0.0.1:8088）

现在访问www.hwcq.online即可成功

服务端配置：

![](../../assets/_images/deploy/nps/nps12.png)

参数配置(可选)
```conf
#HTTP(S) proxy port, no startup if empty
http_proxy_ip=0.0.0.0
http_proxy_port=80      # 20080
https_proxy_port=443    # 20443
https_just_proxy=true
#default https certificate setting
https_default_cert_file=conf/server.pem
https_default_key_file=conf/server.key
```

客户端配置：
```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=123456
auto_reconnection=true
[web]
host=www.hwcq.online
target_addr=127.0.0.1:8088
host_change=www.hwcq.online
header_set_proxy=nps
```

### 2.2 TCP隧道

#### 2.2.1 ssh访问内网

使用场景: 通过公网服务器39.105.58.136的8090端口，连接内网机器192.168.2.3的22端口，实现ssh连接。

服务端配置：

![](../../assets/_images/deploy/nps/nps11.png)

客户端配置：

```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=123456
auto_reconnection=true
[tcp]
mode=tcp
target_addr=127.0.0.1:22
server_port=8090
```

![](../../assets/_images/deploy/nps/nps10.png)


#### 2.2.2 mstsc访问内网

使用场景: 通过公网服务器39.105.58.136的8090端口，连接内网机器192.168.2.3的3389端口

服务端配置：

![](../../assets/_images/deploy/nps/nps5.png)

客户端配置：

```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=123456
auto_reconnection=true
[tcp]
mode=tcp
target_addr=127.0.0.1:3389
server_port=8090
```

![](../../assets/_images/deploy/nps/nps6.png)

![](../../assets/_images/deploy/nps/nps7.png)


#### 2.2.3 http反向代理

服务端配置：

![](../../assets/_images/deploy/nps/nps8.png)

客户端配置：

```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=123456
[tcp]
mode=tcp
target_addr=127.0.0.1:8088
server_port=9090
```

![](../../assets/_images/deploy/nps/nps9.png)


### 2.3 UDP隧道

```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=123456
auto_reconnection=true
[udp]
mode=udp
target_addr=127.0.0.1:8080
server_port=9002
```

### 2.4 HTTP代理

```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=123456
auto_reconnection=true
[http]
mode=httpProxy
server_port=9003
```

### 2.5 SOCKS5代理

使用场景：在外网环境下如同使用vpn一样访问内网设备或者资源，想将公网服务器39.105.58.136的9004端口作为socks5代理，达到访问内网任意设备或者资源的效果

客户端配置：

```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=123456
[socks5]
mode=socks5
server_port=9004
multi_account=multi_account.conf
```

> windows可以配合proxifier进行全局代理。Linux还可以使用proxychains进行socks5配置，推荐linux使用proxychains进行配置，可以更好的联合其他工具进行嗅探收集内网信息和横向移动。

Proxifier是一款功能非常强大的socks5客户端，可以让不支持通过代理服务器工作的网络程序能通过HTTPS或SOCKS代理或代理链。

1. 添加代理服务器

菜单栏点击Proxy Servers图标—add，这里添加socks代理，填写socks服务端的ip和端口

![](../../assets/_images/deploy/nps/nps13.png)

2. 设置代理规则

单击Proxification Rules图标—add，这里设置如果访问172.20.92.*这个ip段则走socks5代理

勾选我们添加的代理规则，默认的代理规勾选为Direct！！！

![](../../assets/_images/deploy/nps/nps14.png)
   
3. 直接访问目标内网的机器

![](../../assets/_images/deploy/nps/nps15.png)


### 2.6 私密代理

tcp隧道暴露了公网vps的监听端口。如果其他人得到了vps的ip，通过端口扫描得到了开放的端口，将会很容易连接上部署的tcp隧道，这是一个很不安全的行为。为了更加安全管理内网设备，nps支持建立私密代理，通过给隧道设置连接密码，增加了隧道的安全性和私密性。攻击机也需要配置nps客户端

服务端配置：

![](../../assets/_images/deploy/nps/nps16.png)

客户端配置：
```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=888888
[secret_ssh]
mode=secret
password=123456
target_addr=127.0.0.1:22
```

攻击机配置：
```bash
./npc -server=39.105.58.136:8024 -vkey=888888 -type=tcp -password=123456 -local_type=secret
ssh -p 2000 root@127.0.0.1
```

如需指定本地端口可加参数`-local_port=xxxx`，默认为2000

### 2.7 P2P代理

适用范围： 大流量传输场景，流量不经过公网服务器，但是由于p2p穿透和nat类型关系较大，不保证100%成功，支持大部分nat类型` ./npc nat -stun_addr=stun.stunprotocol.org:3478`

```bash
vi /etc/nps/conf/nps.conf   # 服务器开启p2p

p2p_ip=39.105.58.136        # 公网ip
p2p_port=6000               # 请在防火墙开放6000~6002(额外添加2个端口)udp端口
```

客户端配置：
```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=999999
[p2p_ssh]
mode=p2p
password=123456
target_addr=127.0.0.1:22
```

攻击机配置：
```bash
./npc -server=39.105.58.136:8024 -vkey=999999 -type=tcp -password=123456 -target=127.0.0.1:22
ssh -p 2000 root@127.0.0.1
```

### 2.8 文件访问

利用nps提供一个公网可访问的本地文件服务，此模式仅客户端使用配置文件模式方可启动

```conf
[common]
server_addr=39.105.58.136:8024
conn_type=tcp
vkey=123456
auto_reconnection=true
[file]
mode=file
server_port=9006
local_path=/tmp/
strip_pre=/web/
```

对于`strip_pre`，访问公网`ip:9006/web/`相当于访问`/tmp/`目录