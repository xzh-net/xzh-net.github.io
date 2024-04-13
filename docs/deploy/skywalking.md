# Apache Skywalking

专门为微服务架构和云原生架构系统而设计并且支持分布式链路追踪的APM（Application Performance Monitor）系统。Apache Skywalking(Incubator)通过加载探针的方式收集应用调用链路信息，并对采集的调用链路信息进行分析，生成应用间关系和服务间关系以及服务指标。SkyWalking的Agent端使用推送模式，OAP服务器端使用拉取模式。

- 官网地址：http://skywalking.apache.org/downloads/
- 历史版本：https://archive.apache.org/dist/skywalking/


## 1. 安装

### 1.1.1 上传解压

```bash
cd /opt/software
tar -zxvf apache-skywalking-apm-9.2.0.tar.gz -C /opt
```

### 1.1.2 修改配置

```bash
mkdir -p /home/elastic/elasticsearch-7.6.2/data /home/elastic/elasticsearch-7.6.2/logs
vi /home/elastic/elasticsearch-7.6.2/config/elasticsearch.yml
```

```conf
cluster.name: elasticsearch               # 默认是elasticsearch
node.name: node-1                         # elasticsearch会默认随机指定一个名字
network.host: 0.0.0.0                     # 允许外网访问
http.port: 9200                           # http访问端口
cluster.initial_master_nodes: ["node-1"]  # 这里就是node.name
path.data: /home/elastic/elasticsearch-7.6.2/data  # 数据目录
path.logs: /home/elastic/elasticsearch-7.6.2/logs  # 日志目录
http.cors.enabled: true                   # head插件跨域
http.cors.allow-origin: "*"
```

### 1.1.3 系统设置

1. 修改文件资源限制

```bash
vi /etc/security/limits.conf
# 末尾添加
elastic soft nofile 65536
elastic hard nofile 65536
```

2. 修改用户线程数

```bash
vi /etc/security/limits.d/20-nproc.conf
# 编辑
elastic soft nofile 65536
elastic hard nofile 65536
* soft nproc 65535
```

3. 修改虚拟内存

```bash
vi /etc/sysctl.conf
# 末尾添加
vm.max_map_count=655360

# 使配置生效
sysctl -p
```

### 1.1.4 创建用户

1. 新增elastic用户

```bash
useradd elastic
passwd elastic   # 为elastic用户设置密码
```

2. 为elastic授权

```bash
chown -R elastic:elastic /home/elastic
```

### 1.1.5 启动服务

```bash
su - elastic
cd /home/elastic/elasticsearch-7.6.2/bin
./elasticsearch -d  # 后台启动
```

### 1.1.6 验证

```bash
ps aux|grep elasticsearch
curl http://127.0.0.1:9200
```


## 1.2 集群

### 1.2.1 拷贝副本

```bash
su - elastic
cp -r elasticsearch-7.6.2   elasticsearch-7.6.2-9201
cp -r elasticsearch-7.6.2   elasticsearch-7.6.2-9202
cp -r elasticsearch-7.6.2   elasticsearch-7.6.2-9203
```

### 1.2.2 节点1

1. 清理数据

```bash
rm -rfv /home/elastic/elasticsearch-7.6.2-9201/data/*
rm -rfv /home/elastic/elasticsearch-7.6.2-9201/logs/*
```

2. 修改配置

```bash
cd /home/elastic/elasticsearch-7.6.2-9201/config
vi elasticsearch.yml
```

配置内容

```conf
cluster.name: elasticsearch
node.name: node-1
node.master: true
node.data: true
network.host: 0.0.0.0
http.port: 9201
transport.tcp.port: 9700
node.max_local_storage_nodes: 3
discovery.seed_hosts: ["localhost:9700","localhost:9800","localhost:9900"]
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
path.data: /home/elastic/elasticsearch-7.6.2-9201/data
path.logs: /home/elastic/elasticsearch-7.6.2-9201/logs
http.cors.enabled: true
http.cors.allow-origin: "*"
```

### 1.2.3 节点2

1. 清理数据

```bash
rm -rfv /home/elastic/elasticsearch-7.6.2-9202/data/*
rm -rfv /home/elastic/elasticsearch-7.6.2-9202/logs/*
```

2. 修改配置

```bash
cd /home/elastic/elasticsearch-7.6.2-9202/config
vi elasticsearch.yml
```

配置内容

```conf
cluster.name: elasticsearch
node.name: node-2
node.master: true
node.data: true
network.host: 0.0.0.0
http.port: 9202
transport.tcp.port: 9800
node.max_local_storage_nodes: 3
discovery.seed_hosts: ["127.0.0.1:9700","127.0.0.1:9800","127.0.0.1:9900"]
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
path.data: /home/elastic/elasticsearch-7.6.2-9202/data
path.logs: /home/elastic/elasticsearch-7.6.2-9202/logs
http.cors.enabled: true
http.cors.allow-origin: "*"
```

### 1.2.4 节点3

1. 清理数据

```bash
rm -rfv /home/elastic/elasticsearch-7.6.2-9203/data/*
rm -rfv /home/elastic/elasticsearch-7.6.2-9203/logs/*
```

2. 修改配置

```bash
cd /home/elastic/elasticsearch-7.6.2-9203/config
vi elasticsearch.yml
```

配置内容

```conf
cluster.name: elasticsearch
node.name: node-3
node.master: true
node.data: true
network.host: 0.0.0.0
http.port: 9203
transport.tcp.port: 9900
node.max_local_storage_nodes: 3
discovery.seed_hosts: ["localhost:9700","localhost:9800","localhost:9900"]
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
path.data: /home/elastic/elasticsearch-7.6.2-9203/data
path.logs: /home/elastic/elasticsearch-7.6.2-9203/logs
http.cors.enabled: true
http.cors.allow-origin: "*"
```

### 1.2.5 启动服务

```bash
su - elastic
/home/elastic/elasticsearch-7.6.2-9201/bin/elasticsearch -d
/home/elastic/elasticsearch-7.6.2-9202/bin/elasticsearch -d
/home/elastic/elasticsearch-7.6.2-9203/bin/elasticsearch -d
```

### 1.2.6 验证集群

http://172.17.17.194:9201/_cat/nodes


### 1.3 插件
