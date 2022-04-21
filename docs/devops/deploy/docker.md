# Docker 18.06.3

## 1. 安装

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum list docker-ce --showduplicates | sort -r
yum install docker-ce
yum install --setopt=obsoletes=0 docker-ce-18.06.3.ce-3.el7 -y
systemctl start docker
chkconfig docker on

curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose -v

#https://hub.docker.com/
```

## 2. 参数设置

1. Docker官方中国区： https://registry.docker-cn.com
2. 网易：http://hub-mirror.c.163.com
3. 中国科学技术大学：https://docker.mirrors.ustc.edu.cn

```Shell
# vi /etc/docker/daemon.json
{
    "registry-mirrors":["https://docker.mirrors.ustc.edu.cn"],
    "insecure-registries": ["192.168.3.200:5000"],
    "exec-opts":["native.cgroupdriver=systemd"],
    "data-root": "/data/docker"
}

mv /var/lib/docker /data # images位置迁移
sudo systemctl daemon-reload 
sudo systemctl restart docker 
```

## 3. 命令
```bash
# 基本信息
docker version              # 查看docker版本
docker info                 # 查看docker信息
docker --help               # 查看docker帮助
docker info | grep Cgroup   # 查看驱动
docker system df            # 查看占用的磁盘空间
docker info | grep "Docker Root Dir"  # 查看镜像位置

# 网络
docker network ls                                     # 查看网络
docker network create -d bridge my-bridge             # 创建一个连接方式是bridge网桥
docker inspect my-bridge                              # 查看my-bridge网络里面的容器
docker network connect my-bridage [containerid]       # 手动将某个容器加入网桥
docker run --name mynginx2 --network my-bridge -p 8080:80 -d nginx:latest # 创建容器时添加到指定网桥中
docker volume create pgdata       # 创建卷
docker volume inspect pgdata      # 查看卷位置
docker run -it -v /[local_path]|pgdata:/[container_path] [imageid] /bin/bash   # 挂载宿主机的一个目录/卷

# 镜像操作
docker images                   # 查看镜像
docker history                  # 查看构建历史
docker rmi [imageid]            # 删除镜像
docker rmi $(docker images -q)  # 删除本地所有镜像
docker rmi $(docker images | grep "none" | awk '{print $3}')    # 删除none的镜像
docker save -o logstash_7.6.2.tar logstash:7.6.2                # 镜像备份
docker load -i logstash_7.6.2.tar                               # 镜像导入
docker tag  serv:1.0 192.168.3.200/xzh/serv:1.1                 # 镜像标记
docker image inspect minio/minio:latest|grep  -i version        # 查看已下载镜像版本

# 容器操作
docker ps -a|-q|-l  
docker start [containerid]
docker start $(docker ps -a -q) 
docker restart [containerid]
docker kill [containerid]
docker stop [containerid]
docker exec -it [containerid] /bin/bash

docker inspect --format '{{ .NetworkSettings.IPAddress }}' [containerid]  # 查看容器ip地址
docker inspect nodered | grep Mounts -A 20                                # 查看容器映射目录
docker rmi -f $(docker images -qa)                                        # 删除所有镜像
docker rm [containerid]                                                   # 删除指定容器
docker rm `docker ps -a -q`                                               # 删除所有容器
docker rm $(docker ps -a | awk '/[imageid]/ {print $1}')        # 删除相同imageid的容器
docker ps -a | grep Exit | cut -d ' ' -f 1 | xargs docker rm    # 删除所有关闭的容器
docker volume rm $(docker volume ls -qf dangling=true)          # 删除所有dangling数据卷(即无用的volume)

docker update --restart=always [container_id]           # 修改指定容器自动启动
docker update --restart=always $(docker ps -q -a)       # 更新所有容器启动时自动启动
docker run --net=host                                   # host模式执行容器，容器端口全部映射给主机
docker run -p 80:80 --name nginx -e TZ="Asia/Shanghai" -d nginx:1.17.0    # 指定容器时区

# 容器日志
docker logs -t --since="2018-02-08T13:23:37" [containerid]          # 查看某时间之后的日志
docker logs -f -t --since="2018-02-08" --tail=100 [containerid]     # 查看指定时间后的日志，只显示最后100行
docker logs --since 30m [containerid]                               # 查看最近30分钟的日志
docker logs -t --since="2018-02-08T13:23:37" --until "2018-02-09T12:23:37" [containerid]  # 查看某时间段日志

# 数据拷贝
docker cp rabbitmq:/[container_path] [local_path]   # 将容器中的文件copy至本地路径
docker cp /etc/localtime [containerid]:/etc/        # 将主机时间到容器
docker cp [local_path] rabbitmq:/[container_path]/  # 将主机文件copy至rabbitmq容器
docker cp [local_path] rabbitmq:/[container_path]   # 将主机文件copy至rabbitmq容器，目录重命名为[container_path]（注意与非重命名copy的区别）

# 镜像分析
docker stats [containerid]                              # 监控指定容器
docker stats $(docker ps -a -q)                         # 监控所有容器
docker stats --no-stream=true $(docker ps -a -q)        # 监控所有容器
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:latest influxdb:1.8
```

## 4. Dockerfile

```bash
FROM        # 设置基础镜像
MAINTAINER  # 设置维护者信息
ADD         # 设置需要添加容器中的文件（自动解压）
COPY        # 设置复制到容器中的文件（不会解压）
USER        # 设置运行RUN指令的用户
ENV         # 设置环境变量可在RUN  指令中使用
RUN         # 设置镜像制作过程中需要运行的指令
ENTRYPOINT  # 设置容器启动时运行的命令(无法覆盖)
CMD         # 设置容器启动时运行的命令(可以覆盖)
WORKDIR     # 设置进入容器的工作目录
EXPOSE      # 设置可被暴露的端口号（端口映射）
VOLUME      # 设置可被挂载的数据卷（目录映射）
ONBUILD     # 设置在构建时需要自动执行的命令
```

>Dockerfile 的指令每执行一次都会在 docker 上新建一层。所以过多无意义的层，会造成镜像膨胀过大

1. 本地构建

- jar

```bash
# 基于openjdk 镜像
FROM openjdk:8-jdk-alpine
ARG JAR_FILE
# 复制文件到容器
COPY ${JAR_FILE} app.jar
# 声明需要暴露的端口
EXPOSE 9001
# 配置容器启动后执行的命令
ENTRYPOINT ["java","-jar","/app.jar"]
```

```shell
# -f :指定要使用的Dockerfile路径；
docker build --build-arg JAR_FILE=sc-gateway.jar -t xzh-gateway:1.0 .
docker images     #查看镜像是否创建成功
docker run -p 9900:9900 --name=xzh-gateway -d xzh-gateway:1.0     #创建容器
```

- 基于debian  

```bash
FROM debian  
COPY . /app  
RUN apt-get update  
RUN apt-get -y install openjdk-11-jdk ssh emacs  
CMD [“java”, “-jar”, “/app/target/my-app-1.0-SNAPSHOT.jar”]  
```

- tomcat8

```bash
FROM centos:7
MAINTAINER xzh xuzhihao@163.com
#拷贝tomcat jdk 到镜像并解压
ADD apache-tomcat-8.5.66.tar.gz /usr/local/tomcat
ADD jdk-8u202-linux-x64.tar.gz /usr/local/jdk
#定义交互时登录路径
ENV MYPATH /usr/local
WORKDIR $MYPATH
#配置jdk 和tomcat环境变量
ENV JAVA_HOME /usr/local/jdk/jdk1.8.0_202
ENV CATALINA_HOME /usr/local/tomcat/apache-tomcat-8.5.66
ENV CATALINA_BASE /usr/local/tomcat/apache-tomcat-8.5.66
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
#设置暴露的端口
EXPOSE 8080
#运行tomcat
CMD /usr/local/tomcat/apache-tomcat-8.5.66/bin/startup.sh && tail -f /usr/local/tomcat/apache-tomcat-8.5.66/logs/catalina.out
```

```shell
docker build -t xzh/tomcat8 .
```

2. 下载构建

[基于ubuntu:14.04的turnserver4.4.2.2](/linux/deploy/docker_hub?id=turn-server-4422)

3. 容器构建

```bash
# 基于当前redis容器创建一个新的镜像；
# -a 提交的镜像作者；
# -c 使用Dockerfile指令来创建镜像；
# -m :提交时的说明文字；
# -p :在commit时，将容器暂停
docker commit -a="xzh" -m="my redis" [redis容器ID]  myredis:v1.1
```

4. 插件构建

pom.xml

```xml
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  <java.version>1.8</java.version>
  <skipTests>true</skipTests>
  <docker.baseImage>openjdk:8-jre-alpine</docker.baseImage>
      <docker.volumes>/tmp</docker.volumes>
      <docker.host>http://172.17.17.148:2375</docker.host>
      <docker.image.prefix>172.17.17.148:88/ec_platform</docker.image.prefix>
      <docker.java.security.egd>-Djava.security.egd=file:/dev/./urandom</docker.java.security.egd>
      <docker.java.opts>-Xms128m -Xmx128m</docker.java.opts>
</properties>

<build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <executions>
					<execution>
						<id>build-image</id>
						<phase>package</phase>
						<goals>
							<goal>build</goal>
						</goals>
					</execution>
				</executions>
                <configuration>
                	<serverId>docker148-harbor88</serverId>
                	<dockerHost>${docker.host}</dockerHost>
                    <imageName>${docker.image.prefix}/${project.artifactId}:${project.version}</imageName>
                    <pushImage>true</pushImage>
                    <baseImage>${docker.baseImage}</baseImage>
                    <volumes>${docker.volumes}</volumes>
                    <env>
                        <JAVA_OPTS>${docker.java.opts}</JAVA_OPTS>
                    </env>
                    <runs>
                    	<run>apk add --update ttf-dejavu fontconfig</run>
                    </runs>
                    <entryPoint>["sh","-c","java $JAVA_OPTS ${docker.java.security.egd} -jar /${project.build.finalName}.jar"]</entryPoint>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>
        <finalName>${project.artifactId}</finalName>
    </build>
```

## 5. Docker Compose

```yml
version: '3'
services:
  mysql:
    image: mysql:5.7
    container_name: mysql
    command: mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root #设置root帐号密码
    ports:
      - 3306:3306
    volumes:
      - /mydata/mysql/data/db:/var/lib/mysql #数据文件挂载
      - /mydata/mysql/data/conf:/etc/mysql/conf.d #配置文件挂载
      - /mydata/mysql/log:/var/log/mysql #日志文件挂载
  redis:
    image: redis:5
    container_name: redis
    command: redis-server --appendonly yes
    volumes:
      - /mydata/redis/data:/data #数据文件挂载
    ports:
      - 6379:6379
  nginx:
    image: nginx:1.10
    container_name: nginx
    volumes:
      - /mydata/nginx/nginx.conf:/etc/nginx/nginx.conf #配置文件挂载
      - /mydata/nginx/html:/usr/share/nginx/html #静态资源根目录挂载
      - /mydata/nginx/log:/var/log/nginx #日志文件挂载
    ports:
      - 80:80
  rabbitmq:
    image: rabbitmq:3.7.15-management
    container_name: rabbitmq
    volumes:
      - /mydata/rabbitmq/data:/var/lib/rabbitmq #数据文件挂载
      - /mydata/rabbitmq/log:/var/log/rabbitmq #日志文件挂载
    ports:
      - 5672:5672
      - 15672:15672
  mongo:
    image: mongo:4.2.5
    container_name: mongo
    volumes:
      - /mydata/mongo/db:/data/db #数据文件挂载
    ports:
      - 27017:27017
  nacos-registry:
    image: nacos/nacos-server:2.0.1
    container_name: nacos-registry
    environment:
      - "MODE=standalone"
    ports:
      - 8848:8848
```

```
docker-compose -f docker-compose-env.yml up -d  
docker-compose -f docker-compose-env.yml down
```



ELK7.6.2.yml

```yml
version: '3'
services:
  elasticsearch:
    image: elasticsearch:7.6.2
    container_name: elasticsearch
    user: root
    environment:
      - "cluster.name=elasticsearch" #设置集群名称为elasticsearch
      - "discovery.type=single-node" #以单一节点模式启动
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" #设置使用jvm内存大小
    volumes:
      - /home/mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins #插件文件挂载
      - /home/mydata/elasticsearch/data:/usr/share/elasticsearch/data #数据文件挂载
    ports:
      - 9200:9200
      - 9300:9300
  kibana:
    image: kibana:7.6.2
    container_name: kibana
    links:
      - elasticsearch:es #可以用es这个域名访问elasticsearch服务
    depends_on:
      - elasticsearch #kibana在elasticsearch启动之后再启动
    environment:
      - "elasticsearch.hosts=http://es:9200" #设置访问elasticsearch的地址
    ports:
      - 5601:5601
  logstash:
    image: logstash:7.6.2
    container_name: logstash
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - /home/mydata/logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf #挂载logstash的配置文件
    depends_on:
      - elasticsearch #kibana在elasticsearch启动之后再启动
    links:
      - elasticsearch:es #可以用es这个域名访问elasticsearch服务
    ports:
      - 4560:4560
      - 4561:4561
      - 4562:4562
      - 4563:4563
```