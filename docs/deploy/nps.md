# NPS v0.26.10

NPS是一款轻量级、高性能、功能强大的内网穿透代理服务器，支持tcp、udp、http等几乎所有流量转发，支持WEB界面管理主机连接。

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

![](../../assets/_images/deploy/nps/nps4.png)

![](../../assets/_images/deploy/nps/nps5.png)

![](../../assets/_images/deploy/nps/nps6.png)

### 1.5 客户端

```bash
npc.exe -server=39.105.58.136 -vkey=tvsel7dmnh6x8c1c -type=tcp
```