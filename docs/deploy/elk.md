# Elastic (ELK) Stack 7.6.2

Elastic Stack是由ELK演化而来，分别是Elasticsearch、logstash、kibana

- 官网地址：https://www.elastic.co/downloads/


## 1. Elasticsearch

### 1.1 单机

#### 1.1.1 上传解压

```bash
cd /opt/software
tar -zxvf elasticsearch-7.6.2-linux-x86_64.tar.gz -C /home/elastic/
```

#### 1.1.2 修改配置

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

#### 1.1.3 系统设置

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

#### 1.1.6 验证

```bash
ps aux|grep elasticsearch
curl http://127.0.0.1:9200
```


### 1.2 集群

#### 1.2.1 拷贝副本

```bash
su - elastic
cp -r elasticsearch-7.6.2   elasticsearch-7.6.2-9201
cp -r elasticsearch-7.6.2   elasticsearch-7.6.2-9202
cp -r elasticsearch-7.6.2   elasticsearch-7.6.2-9203
```

#### 1.2.2 节点1

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

#### 1.2.3 节点2

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

#### 1.2.4 节点3

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

#### 1.2.5 启动服务

```bash
su - elastic
/home/elastic/elasticsearch-7.6.2-9201/bin/elasticsearch -d
/home/elastic/elasticsearch-7.6.2-9202/bin/elasticsearch -d
/home/elastic/elasticsearch-7.6.2-9203/bin/elasticsearch -d
```

#### 1.2.6 验证集群

http://172.17.17.194:9201/_cat/nodes


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

```bash
mkdir -p /home/elastic/elasticsearch-7.6.2/plugins/analysis-ik
cd /home/elastic/elasticsearch-7.6.2/plugins/analysis-ik
wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip
unzip elasticsearch-analysis-ik-7.6.2.zip 
```

2. 重启服务

```bash
ps aux|grep elasticsearch
cd /home/elastic/elasticsearch-7.6.2/bin
./elasticsearch -d
```

## 2. Kibana

### 2.1 下载解压

```bash
cd /opt/software
tar -zxvf kibana-7.6.2-linux-x86_64.tar.gz -C /home/elastic/
cd /home/elastic/
mv kibana-7.6.2-linux-x86_64 kibana-7.6.2
```

### 2.2 目录授权

```bash
chown -R elastic:elastic /home/elastic
```

### 2.3 修改配置

```bash
su - elastic
cd /home/elastic/kibana-7.6.2/config
vi kibana.yml
# 编辑
server.port: 5601
server.host: "0.0.0.0"  # kibana安装服务器
elasticsearch.hosts: ["http://127.0.0.1:9200"]  # elasticsearch服务器，连接集群使用 ，分割
```

### 2.4 启动服务

```bash
cd /home/elastic/kibana-7.6.2/bin
./kibana &
lsof -i:5601
```

### 2.5 客户端测试

访问地址：http://localhost:5601

#### 2.5.1 创建索引

```bash
PUT /mytest/
{
  "settings": {
    "index": {
      "number_of_shards": "3",
      "number_of_replicas": "0"
    }
  }
}
```

#### 2.5.2 查看索引

```bash
GET mytest/_settings
```

#### 2.5.3 删除索引

```bash
DELETE /mytest/
```

#### 2.5.4 添加数据

```bash
POST /mytest/_doc
{
  "title": "我的苹果手机和你的安卓手机，我们都有身份证",
  "images": "http://image.leyou.com/12479122.jpg",
  "price": 598
}
```

#### 2.5.5 排序查询

```bash
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
```

#### 2.5.6 过滤查询

```bash
get mytest/_search
{
  "query": {
    "match": {
      "title": "我们的身份"
    }
  }
}
```

#### 2.5.7 IK分词器

```bash
POST /_analyze
{
  "text": "我的苹果手机和你的安卓手机，我们都有身份证",
  "analyzer": "ik_smart"
}
```

## 3. Logstash

### 3.1 下载解压

```bash
cd /opt/software
tar -zxvf logstash-7.6.2.tar.gz -C /home/elastic/
```

### 3.2 目录授权

```bash
chown -R elastic:elastic /home/elastic
```

### 3.3 修改配置

```bash
su - elastic
cd /home/elastic/logstash-7.6.2/config
cp -p logstash-sample.conf logstash.conf
```

```bash
vi logstash.conf
```

```conf
input {
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4560
    codec => json_lines
    type => "debug"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4561
    codec => json_lines
    type => "error"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4562
    codec => json_lines
    type => "business"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4563
    codec => json_lines
    type => "common"
  }
}
filter{
  if [type] == "common" {
    mutate {
      remove_field => "port"
      remove_field => "host"
      remove_field => "@version"
    }
    json {
      source => "message"
      remove_field => ["message"]
    }
  }
}
output {
  elasticsearch {
    hosts => ["172.17.17.194:9201","172.17.17.194:9202","172.17.17.194:9203"]
    index => "springboot-%{type}-%{+YYYY.MM.dd}"
  }
}
```

### 3.4 安装json_lines插件

```bash
cd /home/elastic/logstash-7.6.2/
vim Gemfile
# 修改source属性为国内的源
source "http://mirrors.tuna.tsinghua.edu.cn/rubygems/"
```

```bash
./bin/logstash-plugin install --no-verify logstash-codec-json_lines
```

### 3.5 启动服务

```bash
cd /home/elastic/logstash-7.6.2/bin
nohup ./logstash -f ../config/logstash.conf &
tail -f ../logs/logstash-plain.log
lsof -i:4560,4561,4562,4563
```

至此，服务端配置完毕，验证应用：https://github.com/xzh-net/spring-boot/tree/main/spring-boot-elk

## 4. Filebeat

### 4.1 下载解压

```bash
cd /opt/software
tar -zxvf filebeat-7.6.2-linux-x86_64.tar.gz -C /home/elastic/
cd /home/elastic/
mv filebeat-7.6.2-linux-x86_64 filebeat-7.6.2
```

### 4.2 目录授权

```bash
chown -R elastic:elastic /home/elastic
```

### 4.3 修改配置

```bash
su - elastic
cd /home/elastic/filebeat-7.6.2
cp -p filebeat.yml filebeat.yml.bak
```

```bash
vi filebeat.yml
```

```conf
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /data/app/logs/application/*/*.log
  exclude_lines: ['\sDEBUG\s\d']
  exclude_files: ['sc-admin.*.log$']
  fields:
    docType: sys-log
    project: microservices-platform
  multiline:
    pattern: '^\[\S+:\S+:\d{2,}] '
    negate: true
    match: after
- type: log
  enabled: true
  paths:
    - /data/app/logs/point/*.log
  fields:
    docType: point-log
    project: microservices-platform
output.logstash:
  hosts: ["172.17.17.194:5044"]
  bulk_max_size: 2048
```

### 4.4 启动服务

```bash
cd /home/elastic/filebeat-7.6.2
./filebeat -c filebeat.yml -e
```


### 4.5 配置规则文件

```bash
mkdir /home/elastic/logstash-7.6.2/patterns
cd /home/elastic/logstash-7.6.2/patterns
vi my_patterns
```

```conf
# user-center
MYAPPNAME [0-9a-zA-Z._-]*
# RMI TCP Connection(2)-127.0.0.1
MYTHREADNAME ([0-9a-zA-Z._-]|\(|\)|\s)*
```


### 4.6 修改logstash

```bash
su - elastic
cd /home/elastic/logstash-7.6.2/config
vi logstash.conf
```

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][docType] == "sys-log" {
    grok {
      patterns_dir => ["patterns"]
      match => { "message" => "\[%{NOTSPACE:appName}:%{IP:serverIp}:%{NOTSPACE:serverPort}\] %{TIMESTAMP_ISO8601:logTime} %{LOGLEVEL:logLevel} %{WORD:pid} \[%{MYAPPNAME:traceId}-%{MYAPPNAME:spanId}\] \[%{MYTHREADNAME:threadName}\] %{NOTSPACE:classname} %{GREEDYDATA:message}" }
      overwrite => ["message"]
    }
    date {
      match => ["logTime","yyyy-MM-dd HH:mm:ss.SSS Z"]
    }
    date {
      match => ["logTime","yyyy-MM-dd HH:mm:ss.SSS"]
      target => "timestamp"
      locale => "en"
      timezone => "+08:00"
    }
    mutate {
      remove_field => "logTime"
      remove_field => "@version"
      remove_field => "host"
      remove_field => "offset"
    }
  }
  if [fields][docType] == "point-log" {
    grok {
      patterns_dir => ["/home/elastic/logstash-7.6.2/patterns"]
      match => {
        "message" => "%{TIMESTAMP_ISO8601:logTime}\|%{MYAPPNAME:appName}\|%{WORD:resouceid}\|%{MYAPPNAME:type}\|%{GREEDYDATA:object}"
      }
    }
    kv {
        source => "object"
        field_split => "&"
        value_split => "="
    }
    date {
      match => ["logTime","yyyy-MM-dd HH:mm:ss.SSS Z"]
    }
    date {
      match => ["logTime","yyyy-MM-dd HH:mm:ss.SSS"]
      target => "timestamp"
      locale => "en"
      timezone => "+08:00"
    }
    mutate {
      remove_field => "message"
      remove_field => "logTime"
      remove_field => "@version"
      remove_field => "host"
      remove_field => "offset"
    }
  }
  if [fields][docType] == "mysqlslowlogs" {
    grok {
        match => [
          "message", "^#\s+User@Host:\s+%{USER:user}\[[^\]]+\]\s+@\s+(?:(?<clienthost>\S*) )?\[(?:%{IP:clientip})?\]\s+Id:\s+%{NUMBER:id}\n# Query_time: %{NUMBER:query_time}\s+Lock_time: %{NUMBER:lock_time}\s+Rows_sent: %{NUMBER:rows_sent}\s+Rows_examined: %{NUMBER:rows_examined}\nuse\s(?<dbname>\w+);\nSET\s+timestamp=%{NUMBER:timestamp_mysql};\n(?<query_str>[\s\S]*)",
          "message", "^#\s+User@Host:\s+%{USER:user}\[[^\]]+\]\s+@\s+(?:(?<clienthost>\S*) )?\[(?:%{IP:clientip})?\]\s+Id:\s+%{NUMBER:id}\n# Query_time: %{NUMBER:query_time}\s+Lock_time: %{NUMBER:lock_time}\s+Rows_sent: %{NUMBER:rows_sent}\s+Rows_examined: %{NUMBER:rows_examined}\nSET\s+timestamp=%{NUMBER:timestamp_mysql};\n(?<query_str>[\s\S]*)",
          "message", "^#\s+User@Host:\s+%{USER:user}\[[^\]]+\]\s+@\s+(?:(?<clienthost>\S*) )?\[(?:%{IP:clientip})?\]\n# Query_time: %{NUMBER:query_time}\s+Lock_time: %{NUMBER:lock_time}\s+Rows_sent: %{NUMBER:rows_sent}\s+Rows_examined: %{NUMBER:rows_examined}\nuse\s(?<dbname>\w+);\nSET\s+timestamp=%{NUMBER:timestamp_mysql};\n(?<query_str>[\s\S]*)",
          "message", "^#\s+User@Host:\s+%{USER:user}\[[^\]]+\]\s+@\s+(?:(?<clienthost>\S*) )?\[(?:%{IP:clientip})?\]\n# Query_time: %{NUMBER:query_time}\s+Lock_time: %{NUMBER:lock_time}\s+Rows_sent: %{NUMBER:rows_sent}\s+Rows_examined: %{NUMBER:rows_examined}\nSET\s+timestamp=%{NUMBER:timestamp_mysql};\n(?<query_str>[\s\S]*)"
        ]
    }
    date {
      match => ["timestamp_mysql","yyyy-MM-dd HH:mm:ss.SSS","UNIX"]
    }
    date {
      match => ["timestamp_mysql","yyyy-MM-dd HH:mm:ss.SSS","UNIX"]
      target => "timestamp"
    }
    mutate {
      convert => ["query_time", "float"]
      convert => ["lock_time", "float"]
      convert => ["rows_sent", "integer"]
      convert => ["rows_examined", "integer"]
      remove_field => "message"
      remove_field => "timestamp_mysql"
      remove_field => "@version"
    }
  }
}

output {
  if [fields][docType] == "sys-log" {
    elasticsearch {
      hosts => ["http://localhost:9200"]
      user => "elastic"
      password => "qEnNfKNujqNrOPD9q5kb"
      index => "sys-log-%{+YYYY.MM.dd}"
    }
  }
  if [fields][docType] == "point-log" {
    elasticsearch {
      hosts => ["http://localhost:9200"]
      user => "elastic"
      password => "qEnNfKNujqNrOPD9q5kb"
      index => "point-log-%{+YYYY.MM.dd}"
      routing => "%{type}"
    }
  }
  if [fields][docType] == "mysqlslowlogs" {
    elasticsearch {
      hosts => ["http://localhost:9200"]
      user => "elastic"
      password => "qEnNfKNujqNrOPD9q5kb"
      index => "mysql-slowlog-%{+YYYY.MM.dd}"
    }
  }
}
```


