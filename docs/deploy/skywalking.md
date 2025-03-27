# Apache Skywalking 8.9.1

专门为微服务架构和云原生架构系统而设计并且支持分布式链路追踪的APM（Application Performance Monitor）系统。Apache Skywalking(Incubator)通过加载探针的方式收集应用调用链路信息，并对采集的调用链路信息进行分析，生成应用间关系和服务间关系以及服务指标。SkyWalking的Agent端使用推送模式，OAP服务器端使用拉取模式。

- 官方网站：http://skywalking.apache.org/downloads/
- APM地址：https://archive.apache.org/dist/skywalking/


## 1. 安装

### 1.1.1 上传解压

```bash
cd /opt/software
wget https://archive.apache.org/dist/skywalking/8.9.1/apache-skywalking-apm-8.9.1.tar.gz
tar -zxvf apache-skywalking-apm-8.9.1.tar.gz -C /usr/local/
mv /usr/local/apache-skywalking-apm-bin /usr/local/skywalking
```

### 1.1.2 修改OAP配置

```bash
cd /usr/local/skywalking/
vi config/application.yml
```

> 使用nacos作为配置中心，需要将core配置中的restHost与gRPCHost修改为本地的ip，默认配置的ip为 0.0.0.0，但如果不修改该ip的话，会导致多个skywalking注册到nacos上时，从nacos管理台界面查看只出现一个skywalking注册成功

```yml
# 集群配置
cluster:
  selector: ${SW_CLUSTER:nacos}
  standalone:
  nacos:
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    hostPort: ${SW_CLUSTER_NACOS_HOST_PORT:192.168.2.201:8848}
    namespace: ${SW_CLUSTER_NACOS_NAMESPACE:"public"}
    username: ${SW_CLUSTER_NACOS_USERNAME:"nacos"}
    password: ${SW_CLUSTER_NACOS_PASSWORD:"nacos"}
    accessKey: ${SW_CLUSTER_NACOS_ACCESSKEY:""}
    secretKey: ${SW_CLUSTER_NACOS_SECRETKEY:""}
# 核心配置
core:
  selector: ${SW_CORE:default}
  default:
    role: ${SW_CORE_ROLE:Mixed}
    restHost: ${SW_CORE_REST_HOST:192.168.2.201}
    restPort: ${SW_CORE_REST_PORT:12800}
    gRPCHost: ${SW_CORE_GRPC_HOST:192.168.2.201}
    gRPCPort: ${SW_CORE_GRPC_PORT:11800}
# 存储配置
storage:
  selector: ${SW_STORAGE:elasticsearch}
  elasticsearch:
    namespace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:192.168.2.201:9200}
```

临时创建镜像，生产环境需手动安装

```bash
docker run --name nacos -e MODE=standalone -p 8848:8848 -d nacos/nacos-server:2.0.1
docker run -d -p 9200:9200 -p 9300:9300 --name es -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms128m -Xmx256m" elasticsearch:7.17.6
```

### 1.1.3 修改UI配置

```bash
cd /usr/local/skywalking/
vi webapp/webapp.yml
```

```yml
server:
  port: 8080

spring:
  cloud:
    gateway:
      routes:
        - id: oap-route
          uri: lb://oap-service
          predicates:
            - Path=/graphql/**
    discovery:
      client:
        simple:
          instances:
            oap-service:
              - uri: http://127.0.0.1:12800
            # - uri: http://<oap-host-1>:<oap-port1>
            # - uri: http://<oap-host-2>:<oap-port2>

  mvc:
    throw-exception-if-no-handler-found: true

  web:
    resources:
      add-mappings: true

management:
  server:
    base-path: /manage
```


### 1.1.4 启动服务

```bash
cd /usr/local/skywalking/bin
./oapService.bat
./webappService.sh
# 或者使用startup.sh启动所有
```

### 1.1.5 客户端

下载探针：https://archive.apache.org/dist/skywalking/java-agent/8.15.0/apache-skywalking-java-agent-8.15.0.tgz

```bash
java -javaagent:skywalking-agent.jar -Dskywalking.agent.service_name=user-center -Dskywalking.collector.backend_service=192.168.2.201:11800,192.168.2.202:11800 -jar user-center.jar
```