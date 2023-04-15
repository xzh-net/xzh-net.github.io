# NPS v0.26.10

NPS是一款轻量级、高性能、功能强大的内网穿透代理服务器。目前支持tcp、udp流量转发，可支持任何tcp、udp上层协议（访问内网网站、本地支付接口调试、ssh访问、远程桌面，内网dns解析等等……），此外还支持内网http代理、内网socks5代理、p2p等，并带有功能强大的web管理端

原理：运行NPS服务的云服务器和运行NPS客户端的主机之间会创建一条TCP或UDP隧道，可以映射云服务器上的某个端口到客户端主机的指定端口，其他主机访问云服务器的这个端口的流量都会通过创建的这条隧道转发到映射的主机端口，实现内网穿透效果

## 1. 安装

### 1.1 下载

![](../../assets/_images/deploy/nps/nps1.png)

下载地址：https://github.com/ehang-io/nps/releases

```bash
mkdir /opt/nps/
cd /opt/nps/
tar -xvf linux_amd64_server.tar.gz
```

### 1.2 修改配置

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

### 1.3 启动服务端

```bash
./nps install       # 输入安装命令
nps start           # 启动nps服务
```

### 1.4 WebUI

访问地址：http://39.105.58.136:8080/

![](../../assets/_images/deploy/nps/nps2.png)

![](../../assets/_images/deploy/nps/nps3.png)

## 2. 使用示例

### 2.1 域名解析

适用范围： 小程序开发、微信公众号开发、产品演示

假设场景：
- 有一个域名hwcq.online，有一台公网机器ip为39.105.58.136
- 两个内网开发站点127.0.0.1:5500，127.0.0.1:5501
- 想通过（http|https://）aaa.hwcq.online访问127.0.0.1:5500，通过（http|https://）bbb.hwcq.online访问127.0.0.1:5501

使用步骤
- 将*.hwcq.online解析到公网服务器39.105.58.136
- 点击刚才创建的客户端的域名管理，添加两条规则规则：1、域名：aaaa.hwcq.online，内网目标：127.0.0.1:5000，2、域名：bbb.hwcq.online，内网目标：127.0.0.1:5001

现在访问（http|https://）aaaa.hwcq.online，bbb.hwcq.online即可成功


### 2.1 通过ssh、mstsc访问内网设备

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

### 2.2 sock5代理访问内网设备

简介：环境同上，通过socks隧道建立连接

### 2.3 http正向代理

```bash
# 被控端启动NPC
npc.exe -server=vpsip:8024 -vkey=123456 -type=tcp
```

![](../../assets/_images/deploy/nps/nps8.png)

![](../../assets/_images/deploy/nps/nps9.png)



### 2.4 私密代理

### 2.5 p2p