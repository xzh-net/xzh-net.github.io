# Fisco Bcos 2.9.1

Fisco Bcos金融区块链合作联盟

## 1. WeBASE一键部署

### 1.1 下载解压

```bash
cd /opt/software
wget https://osp-1257653870.cos.ap-guangzhou.myqcloud.com/WeBASE/releases/download/v3.1.1/webase-deploy.zip
unzip webase-deploy.zip
mv webase-deploy /opt/
```

### 1.2 修改配置

```bash
cd /opt/webase-deploy
vi common.properties
# 修改mysql用户名、密码配置
# 访问：http://localhost:5000
# 默认账号为 admin ，默认密码为 Abcd1234 ，密码不能包含特殊字符
```


## 2. 安装控制台

### 2.1 下载解压

```bash
cd /opt/software
curl -LO https://github.com/FISCO-BCOS/console/releases/download/v2.9.2/download_console.sh && bash download_console.sh
tar -zxvf console.tar.gz -C /opt/fisco
```

### 2.2 修改配置


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

### 2.3 启动服务

```bash
cd /opt/fisco/console && bash start.sh
```