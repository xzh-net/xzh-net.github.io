# NPS v0.26.10

NPS是一款轻量级、高性能、功能强大的内网穿透代理服务器。目前支持tcp、udp流量转发，可支持任何tcp、udp上层协议（访问内网网站、本地支付接口调试、ssh访问、远程桌面，内网dns解析等等……），此外还支持内网http代理、内网socks5代理、p2p等，并带有功能强大的web管理端

原理：运行NPS服务的云服务器和运行NPS客户端的主机之间会创建一条TCP或UDP隧道，可以映射云服务器上的某个端口到客户端主机的指定端口，其他主机访问云服务器的这个端口的流量都会通过创建的这条隧道转发到映射的主机端口，实现内网穿透效果

## 1. 服务端

### 1.1 安装包安装

![](../../assets/_images/deploy/nps/nps1.png)

下载地址：https://github.com/ehang-io/nps/releases

```bash
mkdir /opt/nps/
cd /opt/nps/
tar -xvf linux_amd64_server.tar.gz
```

### 1.2 编译安装

### 1.3 修改配置文件

```bash
cd conf
vi nps.conf
```

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

### 1.4 启动服务端

```bash
./nps install           # 输入安装命令
/etc/nps/conf/nps.conf  # 安装后配置文件位置
nps start               # 启动nps服务
```

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
- 想通过（http|https://www.hwcq.online访问127.0.0.1:8088）

使用步骤：
- 将*.hwcq.online解析到公网服务器39.105.58.136
- 点击刚才创建的客户端的域名管理，添加规则（域名：www.hwcq.online 内网目标：127.0.0.1:8088）

现在访问（http|https://www.hwcq.online即可成功）

客户端配置：
```conf
[common]
server_addr=vpsip:8024
conn_type=tcp
vkey=123456
[web1]
host=www.hwcq.online
target_addr=127.0.0.1:8088
host_change=www.hwcq.online
header_set_proxy=nps
```

![](../../assets/_images/deploy/nps/nps12.png)

服务端配置(可选)
```conf
#HTTP(S) proxy port, no startup if empty
http_proxy_ip=0.0.0.0
http_proxy_port=80      # 修改
https_proxy_port=443    # 修改
https_just_proxy=true
#default https certificate setting
https_default_cert_file=conf/server.pem
https_default_key_file=conf/server.key
```


### 2.1 tcp隧道模式

#### 2.1.1 通过ssh、mstsc访问内网设备

简介：内网设备有暴露可连接的端口，比如22、3389等端口，外网无法访问。NPS安装在有公网IP的环境

```bash
# 被控端启动NPC
npc.exe -server=vpsip:8024 -vkey=123456 -type=tcp
```

`远程桌面配置`

![](../../assets/_images/deploy/nps/nps4.png)

![](../../assets/_images/deploy/nps/nps5.png)

![](../../assets/_images/deploy/nps/nps6.png)

![](../../assets/_images/deploy/nps/nps7.png)

`SSH配置`

![](../../assets/_images/deploy/nps/nps11.png)

![](../../assets/_images/deploy/nps/nps10.png)

#### 2.1.2 http正向代理

```bash
# 被控端启动NPC
npc.exe -server=vpsip:8024 -vkey=123456 -type=tcp
```


![](../../assets/_images/deploy/nps/nps8.png)

![](../../assets/_images/deploy/nps/nps9.png)


### 2.2 udp隧道模式

### 2.3 http代理模式

### 2.4 socks5代理模式

### 2.5 私密代理模式

### 2.6 p2p代理模式

### 2.7 文件访问模式

