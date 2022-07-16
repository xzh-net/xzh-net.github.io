# Elastic (ELK) Stack 7.6.2

- https://www.elastic.co/downloads/elasticsearch
- https://elasticsearch.cn/download/

## 1. Elasticsearch

### 1.1 单机

#### 1.1.1 上传解压

```bash
cd /opt/software
tar -zxvf elasticsearch-7.6.2-linux-x86_64.tar.gz -C /home/elastic/
```

#### 1.1.2 修改es配置

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

#### 1.1.3 修改系统配置

1. 修改文件资源限制

```bash
vi /etc/security/limits.conf
# 末尾添加
elastic soft nofile 65536
elastic hard nofile 65536
*  hard    nproc     4096
```

2. 修改用户线程数

```bash
vi /etc/security/limits.d/20-nproc.conf
# 编辑
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

#### 1.1.4 创建用户

1. 新增elastic用户

```bash
useradd elastic
passwd elastic   # 为elastic用户设置密码
```

2. 为elastic授权

```bash
chown -R elastic:elastic /home/elastic
```

#### 1.1.5 启动服务

```bash
su - elastic
cd /home/elastic/elasticsearch-7.6.2/bin
./elasticsearch -d  # 后台启动
```

#### 1.1.6 验证服务

```bash
ps aux|grep elasticsearch
curl http://127.0.0.1:9200
```


### 1.2 集群

#### 1.2.2 拷贝副本

```bash
su - elastic
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

#### 1.3.1 elasticsearch-head

1. Chrome版

- 下载地址：https://www.crx4chrome.com， 打开搜索`ElasticSearch-Head-0.1.5-Chrome.crx`下载
- 打开Chrome插件设置页面，拖入下载好的.crx插件，启用
- 使用插件：依次点击Chrome右上角插件图标—>ElasticSearch-Head—>输入ES地址—>连接

2. docker版

拉取镜像

```bash
docker run -d --name elasticsearch-head -p 9100:9100 docker.io/mobz/elasticsearch-head:5
```

插件请求406解决：修改容器中`/usr/src/app/_site/vendor.js`文件
- 6886行`application/x-www-form-urlencoded`改成`application/json;charset=UTF-8`
- 7574行`application/x-www-form-urlencoded`改成`application/json;charset=UTF-8`

从容器中拷贝文件到宿主机

```bash
docker cp elasticsearch-head:/usr/src/app/_site/vendor.js /home
docker cp /home/vendor.js elasticsearch-head:/usr/src/app/_site/
docker restart elasticsearch-head
```

#### 1.3.2 analysis-ik

1. 下载解压

https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip

```bash
mkdir -p /home/elastic/elasticsearch-7.6.2/plugins/analysis-ik
cd /home/elastic/elasticsearch-7.6.2/plugins/analysis-ik
# 上传
unzip elasticsearch-analysis-ik-7.6.2.zip 
```

2. 重启服务

```bash
ps aux|grep elasticsearch
cd /home/elastic/elasticsearch-7.6.2/bin
./elasticsearch -d
```



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