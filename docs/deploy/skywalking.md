# Apache Skywalking 8.9.1

专门为微服务架构和云原生架构系统而设计并且支持分布式链路追踪的APM（Application Performance Monitor）系统。Apache Skywalking(Incubator)通过加载探针的方式收集应用调用链路信息，并对采集的调用链路信息进行分析，生成应用间关系和服务间关系以及服务指标。SkyWalking的Agent端使用推送模式，OAP服务器端使用拉取模式。

- 官网地址：http://skywalking.apache.org/downloads/
- APM地址：https://archive.apache.org/dist/skywalking/


## 1. 安装

### 1.1.1 上传解压

```bash
cd /opt/software
wget https://archive.apache.org/dist/skywalking/8.9.1/apache-skywalking-apm-8.9.1.tar.gz
tar -zxvf apache-skywalking-apm-8.9.1.tar.gz -C /usr/local/
mv /usr/local/apache-skywalking-apm-bin /usr/local/skywalking
```

### 1.1.2 集群配置

```bash
cd /usr/local/skywalking/
vi config/application.yml
```

```yml
cluster:
  selector: ${SW_CLUSTER:nacos}
  standalone:
  # Please check your ZooKeeper is 3.5+, However, it is also compatible with ZooKeeper 3.4.x. Replace the ZooKeeper 3.5+
  # library the oap-libs folder with your ZooKeeper 3.4.x library.
  nacos:
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    hostPort: ${SW_CLUSTER_NACOS_HOST_PORT:localhost:8848}
    # Nacos Configuration namespace
    namespace: ${SW_CLUSTER_NACOS_NAMESPACE:"public"}
    # Nacos auth username
    username: ${SW_CLUSTER_NACOS_USERNAME:""}
    password: ${SW_CLUSTER_NACOS_PASSWORD:""}
    # Nacos auth accessKey
    accessKey: ${SW_CLUSTER_NACOS_ACCESSKEY:""}
    secretKey: ${SW_CLUSTER_NACOS_SECRETKEY:""}
```

```bash
docker run --name nacos -e MODE=standalone -p 8848:8848 -d nacos/nacos-server:2.0.1
```

### 1.1.3 存储配置

```bash
cd /usr/local/skywalking/
vi config/application.yml
```

```yml
storage:
  selector: ${SW_STORAGE:elasticsearch}
  elasticsearch:
    namespace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
```

```bash
docker run -d -p 9200:9200 -p 9300:9300 --name es -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms128m -Xmx256m" elasticsearch:7.17.6
```

### 1.1.3 UI配置

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
./startup.sh
./webappService.sh
```

### 1.1.5 客户端

下载探针：https://archive.apache.org/dist/skywalking/java-agent/8.15.0/apache-skywalking-java-agent-8.15.0.tgz

```bash
java -javaagent:skywalking-agent.jar -Dskywalking.agent.service_name=user-center -Dskywalking.collector.backend_service=192.168.2.201:11800 -jar user-center.jar
```