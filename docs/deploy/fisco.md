# Fisco Bcos 2.9.1

FISCO BCOS 平台是金融区块链合作联盟（简称：金链盟）开源工作组以金融业务实践为参考样本，在 BCOS 开源平台基础上进行模块升级与功能重塑，深度定制的安全可控、适用于金融行业且完全开源的区块链底层平台

## 1. 安装

### 1.1 WeBASE

#### 1.1.1 下载解压

```bash
cd /opt/software
wget https://osp-1257653870.cos.ap-guangzhou.myqcloud.com/WeBASE/releases/download/v1.5.5/webase-deploy.zip
unzip webase-deploy.zip
mv webase-deploy /opt/
```

#### 1.1.2 修改配置

```bash
cd /opt/webase-deploy
vi common.properties
```

```conf
[common]
# WeBASE子系统的最新版本（v3.1.0版本）
webase.web.version=v1.5.5
webase.mgr.version=v1.5.5
webase.sign.version=v1.5.5
webase.front.version=v1.5.5

#####################################################################
# if use [installDockerAll] to install WeBASE by docker
# if use [installAll] or [installWeBASE], ignore configuration here

# 1: enable mysql in docker
# 0: mysql run in host, required fill in the configuration of webase-node-mgr and webase-sign
docker.mysql=1

# if [docker.mysql=1], mysql run in host (only works in [installDockerAll])
# run mysql 5.6 by docker
docker.mysql.port=23306
docker.mysql.password=123456
#####################################################################

# 节点管理子系统mysql数据库配置
mysql.ip=localhost
mysql.port=3306
mysql.user=dbUsername
mysql.password=dbPassword
mysql.database=webasenodemanager

# 签名服务子系统mysql数据库配置
sign.mysql.ip=localhost
sign.mysql.port=3306
sign.mysql.user=dbUsername
sign.mysql.password=dbPassword
sign.mysql.database=webasesign

# 节点前置子系统h2数据库名
front.h2.name=webasefront
front.org=fisco

# WeBASE管理平台服务端口
web.port=5000
web.h5.enable=1

# 节点管理子系统服务端口
mgr.port=5001

# 节点前置子系统端口
front.port=5002

# 签名服务子系统端口
sign.port=5004

# 是否使用已有的链（yes/no）
if.exist.fisco=no

# 搭建新链时需配置[if.exist.fisco=no]
# 节点监听Ip
node.listenIp=127.0.0.1
# 节点p2p端口
node.p2pPort=30300
# 节点channel端口
node.channelPort=20200
# 节点rpc端口
node.rpcPort=8545

# SDK连接加密类型 （0: ECDSA SSL, 1: 国密SSL,支持ssl）
encrypt.type=0
encrypt.sslType=0

# FISCO-BCOS版本（v.2.9.1）
fisco.version=2.9.1
# 搭建节点个数（默认两个）
node.counts=nodeCounts

# 使用已有链时需配置[if.exist.fisco=yes]
# 已有链节点rpc端口列表
fisco.dir=/data/app/nodes/127.0.0.1

# 选择fisco.dir下面任意node中的就行（其中包含ca.crt, node.crt and node. Key）
node.dir=node0

```

#### 1.1.3 部署并启动所有服务

```bash
python3 deploy.py installAll
```

> 不要用sudo执行脚本，sudo会导致无法获取当前用户的环境变量如JAVA_HOME。默认账号为 admin ，默认密码为 Abcd1234 ，密码不能包含特殊字符。


```bash
cd /opt/webase-deploy
# 一键部署
python3 deploy.py installAll    # 部署并启动所有服务
python3 deploy.py stopAll       # 停止一键部署的所有服务
python3 deploy.py startAll      # 启动一键部署的所有服务
# 各子服务启停
python3 deploy.py startNode     # 启动FISCO-BCOS节点
python3 deploy.py stopNode      # 停止FISCO-BCOS节点
python3 deploy.py startWeb      # 启动WeBASE-Web
python3 deploy.py stopWeb       # 停止WeBASE-Web
python3 deploy.py startManager  # 启动WeBASE-Node-Manager
python3 deploy.py stopManager   # 停止WeBASE-Node-Manager
python3 deploy.py startSign     # 启动WeBASE-Sign
python3 deploy.py stopSign      # 停止WeBASE-Sign
python3 deploy.py startFront    # 启动WeBASE-Front
python3 deploy.py stopFront     # 停止WeBASE-Front
```


### 1.2 控制台

#### 1.2.1 下载解压

```bash
cd /opt/software
curl -LO https://github.com/FISCO-BCOS/console/releases/download/v2.9.2/download_console.sh && bash download_console.sh
tar -zxvf console.tar.gz -C /opt/fisco
```

#### 1.2.2 修改配置


端口设置，若节点未采用默认端口，请将文件中的20200替换成节点对应的channel端口

```bash
cd /opt/fisco/console
cp -n conf/config-example.toml conf/config.toml
```

配置证书

```bash
cd /opt/fisco/console
cp -r /opt/webase-deploy/nodes/127.0.0.1/sdk/* conf/
```

#### 1.2.3 启动服务

```bash
cd /opt/fisco/console && bash start.sh
```

#### 1.2.4 命令行交互