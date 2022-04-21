# Elastic (ELK) Stack 7.6.2

## 1. Elasticsearch

https://www.elastic.co/downloads/elasticsearch

https://elasticsearch.cn/download/

### 1.1 单机

解压
```bash
tar -zxvf elasticsearch-7.6.2-linux-x86_64.tar.gz -C /opt
```

创建用户
```bash
useradd elsearch      # 新增elsearch用户
passwd elsearch       # 为elsearch用户设置密码
userde1 -r elsearch   # 删除
#为普通用户授权 否则无法运行es
cd /opt/
chown -R elsearch:elsearch elasticsearch-7.6.2
```

修改elasticsearch.yml文件
```bash
vi /opt/elasticsearch-7.6.2/config/elasticsearch.yml
```

```conf
# ================= Elasticsearch   configuration =================
cluster.name: my-application              # 默认是elasticsearch
node.name: node-1                         # elasticsearch会默认随机指定一个名字
network.host: 0.0.0.0                     # 允许外网访问
http.port: 9200                           # http访问端口
cluster.initial_master_nodes: ["node-1"]  # 初始化新的集群时需要此配置来选举master
http.cors.enabled: true                   # head插件跨域
http.cors.allow-origin: "*"
```

修改系统配置文件
```conf
#切换到root用户
su root
vi /etc/security/limits.conf  # 最大可创建文件数太小
# 在文件末尾中增加下面内容
elsearch soft nofile 65536
elsearch hard nofile 65536

vi /etc/security/limits.d/20-nproc.conf # 文件名可能不一致
# 在文件末尾中增加下面内容
elsearch soft nofile 65536
elsearch hard nofile 65536
*  hard    nproc     4096
# 注:*代表Linux所有用户名称

vi /etc/sysctl.conf     # 最大虚拟内存太小
# 在文件中增加下面内容
vm.max_map_count=655360
#重新加载，输入下面命令:
sysctl -p
```

启动elasticsearch
```bash
su - elsearch
cd /opt/elasticsearch-7.6.2/bin
./elasticsearch 
./elasticsearch -d  # 后台启动
```

### 1.2 集群

拷贝副本
```bash
cd /opt
cp -r elasticsearch-7.6.2   elasticsearch-7.6.2-9201
cp -r elasticsearch-7.6.2   elasticsearch-7.6.2-9202
cp -r elasticsearch-7.6.2   elasticsearch-7.6.2-9203
```

修改elasticsearch.yml配置文件
```bash
cd /opt
mkdir logs data
# 授权日志目录
chown -R elsearch:elsearch ./logs ./data
vi /opt/elasticsearch-7.6.2-9201/config/elasticsearch.yml
vi /opt/elasticsearch-7.6.2-9202/config/elasticsearch.yml
vi /opt/elasticsearch-7.6.2-9203/config/elasticsearch.yml
```

9201
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
path.data: /opt/data
path.logs: /opt/logs
http.cors.enabled: true
http.cors.allow-origin: "*"
```

9202
```conf
cluster.name: elasticsearch
node.name: node-2
node.master: true
node.data: true
network.host: 0.0.0.0
http.port: 9202
transport.tcp.port: 9800
node.max_local_storage_nodes: 3
discovery.seed_hosts: ["localhost:9700","localhost:9800","localhost:9900"]
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
path.data: /opt/data
path.logs: /opt/logs
http.cors.enabled: true
http.cors.allow-origin: "*"
```

9203
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
path.data: /opt/data
path.logs: /opt/logs
http.cors.enabled: true
http.cors.allow-origin: "*"
```

执行授权
```bash
在root用户下执行
chown -R elsearch:elsearch /opt/elasticsearch-7.6.2-9201
chown -R elsearch:elsearch /opt/elasticsearch-7.6.2-9202
chown -R elsearch:elsearch /opt/elasticsearch-7.6.2-9203

# 如果有的日志文件授权失败，在root执行
cd /opt/elasticsearch-7.6.2-9201
chown -R elsearch:elsearch logs
cd /opt/elasticsearch-7.6.2-9202
chown -R elsearch:elsearch logs
cd /opt/elasticsearch-7.6.2-9203
chown -R elsearch:elsearch logs
```

启动elasticsearch
```bash
su - elsearch
cd /opt/elasticsearch-7.6.2-9201/bin/
./elasticsearch -d
cd /opt/elasticsearch-7.6.2-9202/bin/
./elasticsearch -d
cd /opt/elasticsearch-7.6.2-9203/bin/
./elasticsearch -d
```


### 1.3 插件

1. elasticsearch-head

- docker安装

```bash
docker run -d --name elasticsearch-head -p 9100:9100 docker.io/mobz/elasticsearch-head:5
```

插件请求406解决：修改容器中`/usr/src/app/_site/vendor.js`文件，6886行`application/x-www-form-urlencoded`改成`application/json;charset=UTF-8`

7574行`application/x-www-form-urlencoded`改成`application/json;charset=UTF-8`

```bash
docker cp elasticsearch-head:/usr/src/app/_site/vendor.js /home
docker cp /home/vendor.js elasticsearch-head:/usr/src/app/_site/
docker restart elasticsearch-head
```

- google插件安装

https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm

2. analysis-ik


## 2. Logstash

## 3. Kibana

```bash

# 创建索引
PUT /mytest/
{
  "settings": {
    "index": {
      "number_of_shards": "3",
      "number_of_replicas": "0"
    }
  }
}

# 查看索引配置
GET mytest/_settings

# 删除索引
DELETE /mytest/

# 添加数据
POST /mytest/_doc
{
  "title": "我的苹果手机和你的安卓手机，我们都有身份证",
  "images": "http://image.leyou.com/12479122.jpg",
  "price": 598
}

# 排序查询
get mytest/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "price": "asc"
    }
  ]
}　

# 过滤查询
get mytest/_search
{
  "query": {
    "match": {
      "title": "我们的身份"
    }
  }
}

# 测试分词器
POST /_analyze
{
  "text": "我的苹果手机和你的安卓手机，我们都有身份证",
  "analyzer": "ik_smart"
}

```

## 4. Filebeat