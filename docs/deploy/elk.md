# Elastic (ELK) Stack 7.6.2

Elastic Stack是由ELK演化而来，分别是Elasticsearch、logstash、kibana

- 官方网站：https://www.elastic.co/downloads/
- 版本一览：https://www.elastic.co/cn/support/matrix#matrix_jvm
- 历史版本：https://www.elastic.co/cn/downloads/past-releases

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

- 下载`ElasticSearch Head 0.1.5.crx`插件
- 打开Chrome浏览器插件管理地址`chrome://extensions/`，拖入下载好的crx，启用
- 如果浏览器提示`无法从该网站添加应用、扩展程序和用户脚本`，将crx文件修改后缀为rar并解压，点击`扩展程序 -> 加载未打包的扩展程序`，选择插件所在目录即可。
- 使用插件：打开Chrome浏览器右上角插件图标，选择ElasticSearch-Head，输入ES地址点击连接。

2. docker版

```bash
# 拉取镜像
docker run -d --name elasticsearch-head -p 9100:9100 docker.io/mobz/elasticsearch-head:5
```

插件请求406解决：修改容器中`/usr/src/app/_site/vendor.js`文件

```bash
# 从容器中拷贝文件到宿主机
docker cp elasticsearch-head:/usr/src/app/_site/vendor.js /home
docker cp /home/vendor.js elasticsearch-head:/usr/src/app/_site/
docker restart elasticsearch-head
```

- 6886行`application/x-www-form-urlencoded`改成`application/json;charset=UTF-8`
- 7573行`application/x-www-form-urlencoded`改成`application/json;charset=UTF-8`


连接es的时候，如果控制台提示跨域，需要修改`elasticsearch.yml`，在文件末尾加入以下配置开启跨域

```yml
http.cors.enabled: true
http.cors.allow-origin: "*"
```


#### 1.3.2 analysis-ik

下载地址：https://release.infinilabs.com/analysis-ik/stable/

1. 离线安装

```bash
mkdir -p /home/elastic/elasticsearch-7.6.2/plugins/analysis-ik
cd /home/elastic/elasticsearch-7.6.2/plugins/analysis-ik
unzip elasticsearch-analysis-ik-7.6.2.zip 
```

2. 在线安装（可选）

```bash
bin/elasticsearch-plugin install https://get.infini.cloud/elasticsearch/analysis-ik/7.6.2
```

3. 重启服务

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

#### 2.5.5 删除数据

删除指定数据
```bash
# 语法格式
DELETE /<index_name>/_doc/<document_id>

DELETE mytest/_doc/1871446175516553218
```

删除所有符合条件的数据，此操作会删除所有匹配的文档，需谨慎操作。
```bash
POST /product/_delete_by_query
{
  "query": {
    "match": {
      "mytest": "身份证"
    }
  }
}
```

如果没有`Kibana Dev Tools`，则使用`curl`命令删除
```bash
curl -X DELETE -u username:password "http://localhost:9200/test/_doc/123"
```


#### 2.5.6 排序查询

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

#### 2.5.7 过滤查询

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

#### 2.5.8 IK分词器

```bash
POST /_analyze
{
  "text": "我的苹果手机和你的安卓手机，我们都有身份证",
  "analyzer": "ik_smart"
}
```

## 3. Logstash

数据流向： `应用 -> Logstash -> Elasticsearch -> Kibana`
 
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
    type => "record"
  }
}
filter{
  if [type] == "record" {
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
    hosts => "172.17.17.161:9200"
    index => "app-%{type}-%{+YYYY.MM.dd}"
  }
}
```

### 3.4 安装json_lines插件

修改下载源

```bash
cd /home/elastic/logstash-7.6.2/
vim Gemfile
# 修改内容
source "http://mirrors.tuna.tsinghua.edu.cn/rubygems/"
```

安装插件

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

### 3.6 客户端测试

创建一个测试脚本

```bash
cd /data/logstash
vi test_logstash.sh
```

```bash
#!/bin/bash

HOST="172.17.17.161"

# 检查nc命令是否可用
check_nc() {
    if ! command -v nc &> /dev/null; then
        echo "错误: nc (netcat) 命令未找到，请先安装"
        exit 1
    fi
}

# 测试连接
test_connection() {
    echo "测试连接到 $HOST..."
    for port in 4560 4561 4562 4563; do
        if nc -z -w 3 $HOST $port; then
            echo "端口 $port: 可达"
        else
            echo "端口 $port: 不可达"
        fi
    done
    echo
}

send_debug() {
    echo "发送 DEBUG 消息..."
    echo '{"message": "Debug test message", "level": "DEBUG", "timestamp": "'$(date -Iseconds)'", "source": "test_script"}' | nc -w 3 $HOST 4560
}

send_error() {
    echo "发送 ERROR 消息..."
    echo '{"message": "Error test message", "level": "ERROR", "timestamp": "'$(date -Iseconds)'", "source": "test_script"}' | nc -w 3 $HOST 4561
}

send_business() {
    echo "发送 BUSINESS 事务..."
    echo '{"message": "Business transaction", "action": "purchase", "amount": 100.50, "timestamp": "'$(date -Iseconds)'", "user_id": "test_user_123"}' | nc -w 3 $HOST 4562
}

send_record() {
    echo "发送 RECORD 数据..."
    # 修正：移除多余的空格和引号问题
    echo '{"record_id": 456, "type": "user", "details": {"name": "john", "age": 30}, "timestamp": "'$(date -Iseconds)'"}' | nc -w 3 $HOST 4563
}

# 主执行函数
main() {
    echo "开始 Logstash 连接测试..."
    echo "目标主机: $HOST"
    echo "======================================"
    
    # 检查依赖
    check_nc
    
    # 测试连接
    test_connection
    
    # 发送测试数据
    send_debug
    sleep 1
    send_error
    sleep 1
    send_business
    sleep 1
    send_record
    
    echo "======================================"
    echo "测试数据发送完成"
    echo "请检查 Logstash 日志和输出目标（如 Elasticsearch）来验证数据接收"
}

# 执行主函数
main
```

给予执行权限
```bash
chmod +x test_logstash.sh
./test_logstash.sh
```

## 4. Filebeat

数据流向： `应用 -> Filebeat  -> Kafka`

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

参考文档：https://www.elastic.co/guide/en/beats/filebeat/7.17/filebeat-input-log.html

```bash
su - elastic
cd /home/elastic/filebeat-7.6.2
cp -p filebeat.yml filebeat.yml.bak
```

```bash
vi filebeat.yml
```

```yml
# 输入配置
filebeat.inputs:
- type: log 
  paths:
    - /var/log/system.log
    - /var/log/wifi.log
- type: log 
  paths:
    - "/var/log/apache2/*"
  fields:
    apache: true
  fields_under_root: true

# 输出配置 Kafka
output.kafka:
  hosts: ["kafka1:9092", "kafka2:9092", "kafka3:9092"]
  # 动态消息主题名称（从字段fields.log_topic获取）
  topic: '%{[fields.log_topic]}'
  partition.round_robin:        # 启用轮询分区策略
    reachable_only: false       # 是否仅选择可访问分区（当前设为false）

  required_acks: 1              # 生产者需要收到的确认数量（1=仅需主分区确认）
  compression: gzip             # 消息压缩算法（gzip格式）
  max_message_bytes: 1000000    # 单条消息最大字节数（约1MB）

# 输出配置 Elasticsearch
output.elasticsearch:
  hosts: ["https://es:9200"]
  username: "filebeat_writer"
  password: "YOUR_PASSWORD"

# 设置kibana
setup.kibana.host: "http://localhost:5601"
setup.dashboards.enabled: true

# 日志配置
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat.log
  keepfiles: 7
  permissions: 0644
```

### 4.4 启动服务

```bash
cd /home/elastic/filebeat-7.6.2
# 查看版本
filebeat version
# 测试配置
filebeat test config -c filebeat.yml
# 测试输出
filebeat test output -c filebeat.yml
# 查看实时参数启动
./filebeat -c filebeat.yml -e
# 导入Kibana仪表盘
./filebeat setup -e
# 后台启动
./filebeat -c filebeat.yml &
```

### 4.6 客户端测试

模拟数据写入

```bash
echo '{"level": "info", "message": "我们都是好孩子"}' >> audit.log
echo '{"我是个大盗贼": {"name": "zhangsan"}}' >> audit.log
```

去Kafka查看消费数据