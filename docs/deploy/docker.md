# Docker 18.06.3

仓库地址：https://hub.docker.com/

## 1. 安装

### 1.1 yum安装

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum install docker-ce
systemctl start docker  
systemctl enable docker
# 查询所有版本，安装指定版本
yum list docker-ce --showduplicates | sort -r   
yum install --setopt=obsoletes=0 docker-ce-18.06.3.ce-3.el7 -y
```

### 1.2 离线安装

下载地址：https://download.docker.com/linux/static/stable/x86_64/

```bash
tar zxf docker-18.06.1-ce.tgz   # 上传解压
sudo cp docker/* /usr/bin/      # 拷贝到全局命令中
sudo dockerd &                  # 启动
```

注册服务

```bash
sudo vi /usr/lib/systemd/system/docker.service
```

```yml
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
 
[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
 
[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload           # 重载守护进程
systemctl start docker
systemctl enable docker
```

### 1.3 RPM安装


```bash
yum install -y yum-plugin-downloadonly
yum install --downloadonly --downloaddir=/data/rpm/ docker-ce-18.06.3.ce-3.el7  # 只下载不安装
cd /data/rpm
yum localinstall *.rpm
```

### 1.4 修改配置

1. 配置镜像加速器、仓库地址、根目录

```bash 
sudo mkdir -p /etc/docker
```

```bash
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://l4ud74lw.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.2.100:88"],
  "exec-opts":["native.cgroupdriver=systemd"],
  "data-root": "/data/docker"
}
EOF
```

2. 开启Remote API

```bash
vim /usr/lib/systemd/system/docker.service
# 添加配置文件内容，-H之前是默认参数，追加 -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
ExecStart=/usr/bin/dockerd  -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock
```

3. 非初始化安装后迁移镜像（可选）

```bash
mv /var/lib/docker /data
```

4. 重启服务

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 1.5 安装docker-compose

```bash
curl -L https://github.com/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
cp /usr/local/bin/docker-compose /usr/bin/docker-compose
docker-compose -v
```

1. mall-env.yml

```bash
vi docker-compose-env.yml
```

```yml
version: '3'
services:
  mysql:
    image: mysql:5.7
    container_name: mysql
    command: mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root     # 设置root帐号密码
    ports:
      - 3306:3306
    volumes:
      - /data/mysql/conf:/etc/mysql/conf.d  # 配置文件挂载
      - /data/mysql/data:/var/lib/mysql     # 数据文件挂载
      - /data/mysql/logs:/logs              # 日志文件挂载
  redis:
    image: redis:5
    container_name: redis
    command: redis-server --appendonly yes
    volumes:
      - /data/redis/data:/data      # 数据文件挂载
    ports:
      - 6379:6379
  nginx:
    image: nginx:1.10
    container_name: nginx
    volumes:
      - /data/nginx/nginx.conf:/etc/nginx/nginx.conf    # 配置文件挂载
      - /data/nginx/html:/usr/share/nginx/html          # 静态资源根目录挂载
      - /data/nginx/log:/var/log/nginx                  # 日志文件挂载
    ports:
      - 80:80
  rabbitmq:
    image: rabbitmq:3.7.15-management
    container_name: rabbitmq
    volumes:
      - /data/rabbitmq/data:/var/lib/rabbitmq   # 数据文件挂载
      - /data/rabbitmq/log:/var/log/rabbitmq    # 日志文件挂载
    ports:
      - 5672:5672
      - 15672:15672
  mongo:
    image: mongo:4.2.5
    container_name: mongo
    volumes:
      - /data/mongo/db:/data/db     # 数据文件挂载
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


2. elk-762.yml

```bash
vi docker-compose-elk.yml
```

```yml
version: '3'
networks:
  es:
services:
  elasticsearch:
    image: elasticsearch:7.6.2
    container_name: elasticsearch
    user: root
    environment:
      - "cluster.name=elasticsearch"      # 设置集群名称为elasticsearch
      - "discovery.type=single-node"      # 以单一节点模式启动
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"  # 设置使用jvm内存大小
    volumes:
      - /data/elasticsearch/plugins:/usr/share/elasticsearch/plugins  # 插件文件挂载
      - /data/elasticsearch/data:/usr/share/elasticsearch/data        # 数据文件挂载
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      - es  
  kibana:
    image: kibana:7.6.2
    container_name: kibana
    links:
      - elasticsearch:es  # 可以用es这个域名访问elasticsearch服务
    depends_on:
      - elasticsearch     # kibana在elasticsearch启动之后再启动
    environment:
      - "elasticsearch.hosts=http://es:9200"  # 设置访问elasticsearch的地址
    ports:
      - 5601:5601
    networks:
      - es  
  logstash:
    image: logstash:7.6.2
    container_name: logstash
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - /data/logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf     # 挂载logstash的配置文件
    depends_on:
      - elasticsearch       # kibana在elasticsearch启动之后再启动
    links:
      - elasticsearch:es    # 可以用es这个域名访问elasticsearch服务
    ports:
      - 4560:4560
      - 4561:4561
      - 4562:4562
      - 4563:4563
    networks:
      - es  
```

3. spark.yml

```bash
vi spark.yml
```

```yml
version: '3.8'

services:
  spark-master:
    image: bde2020/spark-master:3.2.0-hadoop3.2
    container_name: spark-master
    ports:
      - "8080:8080"
      - "7077:7077"
    volumes:
      - /home/data/spark:/data
    environment:
      - INIT_DAEMON_STEP=setup_spark
  spark-worker-1:
    image: bde2020/spark-worker:3.2.0-hadoop3.2
    container_name: spark-worker-1
    depends_on:
      - spark-master
    ports:
      - "8081:8081"
    volumes:
      - /home/data/spark:/data
    environment:
      - "SPARK_MASTER=spark://spark-master:7077"
  spark-worker-2:
    image: bde2020/spark-worker:3.2.0-hadoop3.2
    container_name: spark-worker-2
    depends_on:
      - spark-master
    ports:
      - "8082:8081"
    volumes:
      - /home/data/spark:/data
    environment:
      - "SPARK_MASTER=spark://spark-master:7077"
```

启动

```bash
docker-compose -f spark.yml up -d
docker exec -it [容器的id] /bin/bash
ls /spark/bin
/spark/bin/spark-shell --master spark://spark-master:7077 --total-executor-cores 8 --executor-memory 2560m
```

访问地址：http://127.0.0.1:8080

```bash
val rdd=sc.parallelize(Array(1,2,3,4,5,6,7,8))  # 创建一个RDD
rdd.collect()                                   # 打印rdd内容
rdd.partitions.size                             # 查询分区数
val rddFilter=rdd.filter(_ > 5)                 # 选出大于5的数值
rddFilter.collect()                             # 打印rddFilter内容
:quit                                           # 退出spark-shell
```

### 1.6 卸载

```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
rm -rf /var/lib/docker
```


## 2. 命令

### 2.1 基本

```bash
docker version              # 查看docker版本
docker info                 # 查看docker信息
docker --help               # 查看docker帮助
docker info | grep Cgroup   # 查看驱动
docker system df            # 查看占用的磁盘空间
docker info | grep "Docker Root Dir"  # 查看镜像位置
```

### 2.2 网络

```bash
docker network ls                                       # 查看网络
docker network create -d bridge my-bridge               # 创建一个连接方式是bridge网桥
docker inspect my-bridge                                # 查看my-bridge网络里面的容器
docker network connect my-bridage [containerid]         # 手动将某个容器加入网桥
docker run --name mynginx2 --network my-bridge -p 8080:80 -d nginx:latest   # 创建容器时添加到指定网桥中
```

### 2.3 存储

```bash
docker volume ls
docker volume create pgdata       # 创建卷
docker volume inspect pgdata      # 查看卷位置
docker volume rm $(docker volume ls -qf dangling=true)          # 删除所有dangling数据卷(即无用的volume)
docker run -it -v /[local_path]|pgdata:/[container_path] [imageid] /bin/bash   # 挂载宿主机的一个目录/卷
```

### 2.4 镜像

```bash
docker images                   # 查看镜像
docker history                  # 查看构建历史
docker rmi [imageid]                                                    # 删除指定镜像
docker rmi $(docker images | grep "none" | awk '{print $3}')            # 删除none的镜像
docker rmi $(docker images -qa)                                         # 删除所有镜像
docker save -o logstash_7.6.2.tar logstash:7.6.2                # 镜像备份
docker load -i logstash_7.6.2.tar                               # 镜像导入
docker tag  serv:1.0 192.168.3.200/xzh/serv:1.1                 # 镜像标记
docker image inspect minio/minio:latest|grep  -i version        # 查看已下载镜像版本
# 镜像分析
docker stats [containerid]                              # 监控指定容器
docker stats $(docker ps -a -q)                         # 监控所有容器
docker stats --no-stream=true $(docker ps -a -q)        # 监控所有容器
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:latest influxdb:1.8
```

> 2024年6月6日，镜像仓库地址无论换成任何第三方全部无法下载，只有阿里云个人镜像容器服务`暂时`可用，让我产生了导出镜像的想法，把平时经常用到的镜像全部离线保存下来，在遇到无法下载时候可以快速搭建调试环境。

镜像导出
```bash
#!/bin/bash

# 获取所有的镜像ID
image_ids=$(docker images -q)
# 循环遍历每个镜像ID
for id in $image_ids; do
    # 获取镜像的名字和版本号
    image_name=$(docker inspect --format='{{ .RepoTags }}' $id | tr -d '[]')
    image_tar=$(echo $image_name | sed 's/\//_/g')
    # 导出镜像为tar文件
    output_file="${image_tar}.tar"
    docker save -o $output_file $image_name
    echo "导出镜像 $image_name 为 $output_file"
done
```

批量导入
```bash
ls *.tar | xargs -I {} docker load -i {}
for t in *.tar; do docker load -i "$t"; done
```

### 2.5 容器

```bash
docker ps -a|-q|-l  
docker start [containerid]
docker start $(docker ps -a -q) 
docker restart [containerid]
docker kill [containerid]
docker stop [containerid]
docker exec -it [containerid] /bin/bash

docker inspect --format '{{ .NetworkSettings.IPAddress }}' [containerid]    # 查看容器ip地址
docker inspect nodered | grep Mounts -A 20                                  # 查看容器映射目录
docker update --restart=always [container_id]                               # 修改指定容器自动启动
docker update --restart=always $(docker ps -q -a)                           # 更新所有容器启动时自动启动
# 删除
docker rm [containerid]                                                     # 删除指定容器
docker rm $(docker ps -a | awk '/[imageid]/ {print $1}')                    # 删除相同imageid的容器
docker ps -a | grep Exit | cut -d ' ' -f 1 | xargs docker rm                # 删除所有关闭的容器
docker rm -f `docker ps -a -q`                                              # 强制删除所有容器
# 日志
docker logs -t --since="2018-02-08T13:23:37" [containerid]                  # 查看某时间之后的日志
docker logs -f -t --since="2018-02-08" --tail=100 [containerid]             # 查看指定时间后的日志，只显示最后100行
docker logs --since 30m [containerid]                                       # 查看最近30分钟的日志
docker logs -t --since="2018-02-08T13:23:37" --until "2018-02-09T12:23:37" [containerid]  # 查看某时间段日志
# 数据拷贝
docker cp rabbitmq:/[container_path] [local_path]   # 将容器中的文件copy至本地路径
docker cp /etc/localtime [containerid]:/etc/        # 将主机时间到容器
docker cp [local_path] rabbitmq:/[container_path]/  # 将主机文件copy至rabbitmq容器
docker cp [local_path] rabbitmq:/[container_path]   # 将主机文件copy至rabbitmq容器，目录重命名为[container_path]（注意与非重命名copy的区别）
```

## 3. 构建

### 3.1 Dockerfile

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

> Dockerfile 的指令每执行一次都会在 docker 上新建一层。所以过多无意义的层，会造成镜像膨胀过大

#### 3.1.1 基于环境构建Jar

1. 创建文件

```bash
cd /data/apps
vi eureka-server.Dockerfile
```

```bash
FROM openjdk:8-jre-alpine
WORKDIR /apps
COPY eureka /apps/
EXPOSE 9001
ENTRYPOINT ["java","-jar","eureka-server.jar"]
```

2. 构建镜像

```bash
docker build -f eureka-server.Dockerfile -t eureka-server .
```

3. 运行

```bash
docker run -dit --name eureka-server -p 9001:9001 eureka-server
docker exec -it eureka-server /bin/ash      # 进入容器
cat /etc/issue                              # 查看操作系统
uname -a                                    # 查看内核
```

#### 3.1.2 基于操作系统构建Tomcat

1. 创建文件

```bash
vi Dockerfile
```

```bash
FROM centos:7
MAINTAINER xzh xzh@163.com
# 拷贝tomcat jdk 到镜像并解压
ADD apache-tomcat-8.5.61.tar.gz /opt
ADD jdk-8u202-linux-x64.tar.gz /usr/local
# 定义交互时登录路径
ENV MYPATH /opt/apache-tomcat-8.5.61
WORKDIR $MYPATH
# 配置jdk 和tomcat环境变量
ENV JAVA_HOME /usr/local/jdk1.8.0_202
ENV CATALINA_HOME /opt/apache-tomcat-8.5.61
ENV CATALINA_BASE /opt/apache-tomcat-8.5.61
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
# 设置暴露的端口
EXPOSE 8080
# 运行tomcat
CMD /opt/apache-tomcat-8.5.61/bin/startup.sh && tail -f /opt/apache-tomcat-8.5.61/logs/catalina.out
```

2. 构建镜像

```bash
docker build -t tomcat8 .
```

3. 运行

```bash
docker run -dit --name tomcat8 -p 8080:8080 tomcat8
docker exec -it tomcat8 /bin/bash
```

#### 3.1.3 turnserver

1. 创建文件

```bash
vi Dockerfile
```

```bash
# Coturn
#
# VERSION 4.4.2.2
FROM  ubuntu:14.04
MAINTAINER xzh<xzh@gmail.com>

RUN apt-get update && apt-get install -y \
  curl \
  libevent-core-2.0-5 \
  libevent-extra-2.0-5 \
  libevent-openssl-2.0-5 \
  libevent-pthreads-2.0-5 \
  libhiredis0.10 \
  libmysqlclient18 \
  libpq5 \
  telnet \
  wget

RUN wget http://turnserver.open-sys.org/downloads/v4.4.2.2/turnserver-4.4.2.2-debian-wheezy-ubuntu-mint-x86-64bits.tar.gz \
  && tar xzf turnserver-4.4.2.2-debian-wheezy-ubuntu-mint-x86-64bits.tar.gz \
  && dpkg -i coturn_4.4.2.2-1_amd64.deb

COPY ./turnserver.sh /turnserver.sh

ENV TURN_USERNAME kurento
ENV TURN_PASSWORD kurento
ENV REALM kurento.org
ENV NAT true

EXPOSE 3478 3478/udp

ENTRYPOINT ["/turnserver.sh"]
```

2. 编写启动脚本

```bash
vim turnserver.sh
```

```bash
#!/bin/bash
set -e

if [ $NAT = "true" -a -z "$EXTERNAL_IP" ]; then

  # Try to get public IP
  PUBLIC_IP=$(curl http://169.254.169.254/latest/meta-data/public-ipv4) || echo "No public ip found on http://169.254.169.254/latest/meta-data/public-ipv4"
  if [ -z "$PUBLIC_IP" ]; then
    PUBLIC_IP=$(curl http://icanhazip.com) || exit 1
  fi

  # Try to get private IP
  PRIVATE_IP=$(ifconfig | awk '/inet addr/{print substr($2,6)}' | grep -v 127.0.0.1) || exit 1
  export EXTERNAL_IP="$PUBLIC_IP/$PRIVATE_IP"
  echo "Starting turn server with external IP: $EXTERNAL_IP"
fi

echo 'min-port=49152' > /etc/turnserver.conf
echo 'max-port=65535' >> /etc/turnserver.conf
echo 'fingerprint' >> /etc/turnserver.conf
echo 'lt-cred-mech' >> /etc/turnserver.conf
echo "realm=$REALM" >> /etc/turnserver.conf
echo 'log-file stdout' >> /etc/turnserver.conf
echo "user=$TURN_USERNAME:$TURN_PASSWORD" >> /etc/turnserver.conf
[ $NAT = "true" ] && echo "external-ip=$EXTERNAL_IP" >> /etc/turnserver.conf

exec /usr/bin/turnserver "$@"
```


3. 构建镜像

```bash
chmod -R 777 /home/coturn
sudo docker build --tag coturn .
sudo docker run -p 3478:3478 -p 3478:3478/udp coturn
```

#### 3.1.4 hadoop 3.x

1. 构建centos7-ssh-sync

```bash
cd /data/dockerfile/centos7-ssh-sync
vi Dockerfile
```

```bash
FROM centos:7
MAINTAINER xzh

ENV TZ=Asia/Shanghai
#install vim & net-tools
RUN yum -y install vim
RUN yum -y install net-tools

#install rsync
RUN yum -y install rsync

#安装ssh
RUN yum install -y openssh-server sudo
RUN sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config
RUN yum  install -y openssh-clients

#配置用户名和密码
RUN echo "root:123456" | chpasswd
RUN echo "root   ALL=(ALL)       ALL" >> /etc/sudoers
#生成ssh key
RUN ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key

#配置sshd服务
RUN mkdir /var/run/sshd
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

```bash
docker build -f Dockerfile -t centos7-ssh-sync .
```

2. 构建centos7-hadoop

```bash
cd /data/dockerfile/centos7-hadoop3
vi Dockerfile
```

```bash
FROM centos7-ssh-sync
MAINTAINER xzh

# install jdk8
ADD jdk-8u202-linux-x64.tar.gz /usr/local/
ENV JAVA_HOME /usr/local/jdk1.8.0_202
ENV PATH $PATH:$JAVA_HOME/bin

# install hadoop3.1.4
ADD hadoop-3.1.4-bin-snappy-CentOS7.tar.gz /usr/local/
ENV HADOOP_HOME /usr/local/hadoop-3.1.4
ENV PATH $PATH:$HADOOP_HOME/bin
ENV PATH $PATH:$HADOOP_HOME/sbin

WORKDIR /usr/local
```

```bash
docker build -f Dockerfile -t centos7-hadoop3 .
```

3. 运行

```bash
docker run -dit --name hadoop3 -h hadoop3 -p 1022:22 \
-p 8020:8020 -p 9870:9870 -p 9871:9871 \
-p 9866:9866 -p 9864:9864 -p 9865:9865 \
-p 8088:8088 \
-p 14000:14000 --restart=always --privileged=true centos7-hadoop3
```

4. 登录容器

```bash
docker exec -it hadoop3 /bin/bash
hadoop version
java -version
```

5. 设置hadoop-env.sh

```bash
cd /usr/local/hadoop-3.1.4/etc/hadoop
vi hadoop-env.sh 
```

```bash
# 添加下面这些内容
export JAVA_HOME=/usr/local/jdk1.8.0_202
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root 
```

6. 设置core-site.xml

```bash
vi core-site.xml
```

```yaml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop3:8020</value>
    </property>
    <property>
        <name>hadoop.http.staticuser.user</name>
        <value>root</value>
    </property>
    <property>
        <name>dfs.permissions.enabled</name>
        <value>false</value>
    </property>
</configuration>
```

7. 设置hdfs-site.xml

```bash
vi hdfs-site.xml
```

```yaml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

8. 启动服务

```bash
ssh-keygen
ssh-copy-id hadoop3
hdfs namenode -format
start-all.sh
jps
```

访问地址：http://ip:9870

> web上传附件，需要在本地hosts添加 `0.0.0.0 hadoop3`，否则提示`Couldn't upload the file`问题

### 3.2 容器构建

基于一个已存在的容器构建出镜像
   - -a 提交的镜像作者；
   - -c 使用Dockerfile指令来创建镜像；
   - -m :提交时的说明文字；

```bash
docker commit  -a="xzh" -m="my container" [containerid]  container:v1.1
```

### 3.3 插件构建

基于 `io.fabric8` 构建，强烈推荐

```xml
<plugin>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.44.0</version>
    <configuration>
        <!--指定远程服务器的Docker服务访问地址 -->
        <dockerHost>tcp://192.168.2.100:2375</dockerHost>
        <!--指定私有仓库的访问路径 -->
        <pushRegistry>http://192.168.2.100:88</pushRegistry>
        <!--指定私有仓库的用户名与密码 -->
        <authConfig>
            <username>admin</username>
            <password>Harbor12345</password>
        </authConfig>
        <images>
            <image>
                <!--指定私有仓库访问地址/项目名称/镜像名称/版本 -->
                <name>192.168.2.100/ec_platform/${project.artifactId}:${project.version}</name>
                <build>
                    <!--指定Dockerfile的路径 -->
                    <dockerFileDir>${project.basedir}</dockerFileDir>
                </build>
            </image>
        </images>
    </configuration>
    <!--指定在每次打包或者重新打包的时候运行该插件的build, push, run目标 -->
    <executions>
        <execution>
            <id>build-image</id>
            <phase>package</phase>
            <goals>
                <goal>build</goal>
                <goal>push</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

创建Dockerfile

```bash
FROM openjdk:8-jre-alpine
COPY target/*.jar /app.jar
EXPOSE 8080
ENTRYPOINT ["sh","-c","java -Xms128m -Xmx128m -Djava.security.egd=file:/dev/./urandom -jar /app.jar"]
```

## 4. 仓库

### 4.1 容器

#### 4.1.1 portainer-ce

```bash
docker volume create portainer_data

docker run -d -p 8000:8000 -p 9443:9443 --name portainer \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:2.11.0
```

#### 4.1.2 Nginx

```bash
docker run -p 80:80 --name nginx \
-v /data/nginx/html:/usr/share/nginx/html \
-v /data/nginx/logs:/var/log/nginx  \
-d nginx:1.20.2
```

```bash
docker container cp nginx:/etc/nginx /data/nginx/   # 将容器内的配置文件拷贝到指定目录
mv /data/nginx/nginx /data/nginx/conf               # 修改文件
docker rm -f nginx
```

启动服务

```bash
docker run -p 80:80 -p 443:443 --name nginx \
-v /data/nginx/html:/usr/share/nginx/html \
-v /data/nginx/logs:/var/log/nginx  \
-v /data/nginx/conf:/etc/nginx \
-d nginx:1.20.2
```

#### 4.1.3 Nginx-fancyindex

```bash
docker run -d \
  -p 8083:80 \
  -p 8084:443 \
  -e HTTP_AUTH="off" \
  -e HTTP_USERNAME="admin" \
  -e HTTP_PASSWD="admin" \
  -v /home/my-files:/app/public \
  --restart unless-stopped \
  --mount type=tmpfs,destination=/tmp \
  80x86/nginx-fancyindex
```

#### 4.1.4 Rancher

```bash
mkdir -p /data/rancher_home/rancher
mkdir -p /data/rancher_home/auditlog

docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 \
  -v /data/rancher_home/rancher:/var/lib/rancher \
  -v /data/rancher_home/auditlog:/var/log/auditlog \
  --name rancher rancher/rancher 
```

默认安装密码 `docker logs rancher 2>&1 | grep "Bootstrap Password:"`，重置密码 `docker exec -it rancher reset-password`

### 4.2 关系型数据库

#### 4.2.1 MySQL


```bash
mkdir -p /data/mysql/data /data/mysql/logs /data/mysql/conf

docker run -p 3306:3306 --name mysql \
-v /data/mysql/conf:/etc/mysql/conf.d \
-v /data/mysql/data:/var/lib/mysql \
-v /data/mysql/logs:/logs \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7

docker cp /data/mall.sql mysql:/  # sql拷贝到容器中
docker exec -it mysql /bin/bash
mysql -uroot -proot --default-character-set=utf8
```

```sql
create database mall character set utf8
use mall;
source /mall.sql;
grant all privileges on *.* to 'root' @'%' identified by '123456';
```

#### 4.2.2 PostgreSQL

```bash
docker volume create pgdata  # 创建本地卷，数据卷可以在容器之间共享和重用，默认会一直存在，即使容器被删除
docker volume inspect pgdata # 查看数据卷的本地位置

docker run  -p 5432:5432 --name postgres2 \
    -e POSTGRES_USER=postgres \
    -e POSTGRES_PASSWORD=123456 \
    -e POSTGRES_DB=testdb \
    -e TZ=PRC  \
    -v pgdata:/var/lib/postgresql/data \
    -d postgres:12.4

docker exec -it postgres2 /bin/bash
/var/lib/postgresql/data   #  镜像的data目录
/usr/lib/postgresql/12/bin #  工具目录
psql -Upostgres # 连接数据库
```

#### 4.2.3 Oracle

```bash
docker pull wnameless/oracle-xe-11g
docker pull wnameless/oracle-xe-11g-r2

docker run --name oracle-xe-11g -d -v /data/oracle_data:/data/oracle_data -p 49160:22 -p 49161:1521 -e ORACLE_ALLOW_REMOTE=true wnameless/oracle-xe-11g:14.04.4

# 连接参数
port: 49161
sid: xe
username: system
password: oracle
system: oracle
sys: oracle

# 进入容器
docker exec -it oracle-xe-11g /bin/bash

cd /u01/app/oracle
mkdir oradata
chmod 777 oradata
su oracle
cd $ORACLE_HOME
bin/sqlplus / as sysdba

# 创建表空间
create tablespace xzh datafile '/u01/app/oracle/oradata/xzh.dbf' size 100M;
create user xzh0610 identified by 123456 default tablespace xzh;
grant connect,resource to xzh0610;
grant dba to xzh0610;  #  授予dba权限后，这个用户能操作所有用户的表
```

#### 4.2.4 SQL Server

```bash
docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=Pass@w0rd" \
   -p 1433:1433 --name mssql -h mssql \
   -d mcr.microsoft.com/mssql/server:2019-latest

docker exec -it mssql "bash"
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P "Pass@w0rd"
```

```sql
create database datax
go
create table pms_product (
    id int,
    name varchar(100),   
    brand_name varchar(100),  
    create_time datetime
)
go
insert into pms_product values (1,'nnn','xxx',getdate())
go
select * from pms_product
go
```

#### 4.2.5 DM

请在达梦数据库官网下载 [Docker](https://eco.dameng.com/download/) 安装包，拷贝安装包到 `/opt` 目录下，执行以下命令导入安装包：

```bash
docker load -i dm8_20220822_rev166351_x86_rh6_64_ctm.tar
docker run -d -p 5236:5236 --restart=always --name dm8_01 --privileged=true -e PAGE_SIZE=16 -e LD_LIBRARY_PATH=/opt/dmdbms/bin -e INSTANCE_NAME=dm8_01 -v /data/dm8_01:/opt/dmdbms/data dm8_single:v8.1.2.128_ent_x86_64_ctm_pack4
```

> 注意:  
> 1.如果使用 docker 容器里面的 disql，进入容器后，先执行 source /etc/profile 防止中文乱码。  
> 2.新版本 Docker 镜像中数据库默认用户名/密码为 SYSDBA/SYSDBA001  

### 4.3 非关系型数据库

#### 4.3.1 Memcached

```bash
docker pull memcached:1.6.12
docker run -dit --name memcached -m 128m -c 16382 -p 11211:11211 -d memcached:1.6.12
```

可视化
```bash
docker run -dit -p 8080:8080 -e MEMADMIN_USERNAME='admin' -e MEMADMIN_PASSWORD='admin' -e MEMCACHED_HOST='172.17.17.200' -e MEMCACHED_PORT='11211' kitsudo/memadmin:latest
# 启动后报错进入容器，文件最后添加一行
vim /etc/apache2/apache2.conf
ServerName localhost:80
# 然后保存并重启Apache2
service apache2 restart
```

#### 4.3.2 Redis

- 单机

```bash
docker run -p 6379:6379 --name redis \
-v /data/redis/data:/data \
-d redis:5 redis-server --appendonly yes --requirepass "123456"
```

- 布隆过滤器（6.0.9）

```bash
docker run -dit -p 6379:6379 --name redis-redisbloom -d --restart=always -e TZ="Asia/Shanghai" redislabs/rebloom:2.2.8 --requirepass  "redis6379"
```

- 监控

```bash
docker run --name redis-stat --link some-redis:redis -p 8080:63790 -d insready/redis-stat --server redis          # 容器内部自连接
docker run --name redis-stat -p 8080:63790 -d insready/redis-stat --server 192.168.3.200:6379                     # 远程服务器
docker run --name redis-stat -p 8080:63790 -d insready/redis-stat --server 192.168.3.200:6379 192.168.3.201:6379  # 远程集群或单机
```

- prometheus监控

```bash
docker pull oliver006/redis_exporter:v1.28.0
docker run -d --name redis_exporter16379 -p 16379:9121 oliver006/redis_exporter:v1.28.0 --redis.addr redis://172.17.17.191:16379 --redis.password 'redis16379'
```

#### 4.3.3 MongoDB

```bash
docker pull mongo:4.4.6

docker run -p 27017:27017 --name mongo \
-v /data/mongo/db:/data/db \
-d mongo:4.4.6
```

#### 4.3.4 CouchDB 

```bash
docker run -p 5984:5984 --name my-couchdb -e COUCHDB_USER=admin -e COUCHDB_PASSWORD=admin -v /data/couchdb/data:/opt/couchdb/data -d couchdb:3.2
docker run -d -p 8800:8000 --link=my-couchdb --name fauxton 3apaxicom/fauxton sh -c 'fauxton -c http://192.168.3.200:5984'  # 可视化
```

#### 4.3.5 InfluxDB

- 1.8

```bash
docker run -d -p 8086:8086 \
      -v /data/influxdb1:/var/lib/influxdb \
      --name influxdb1 \
      influxdb:1.8
```

- 2.0.6

```bash
docker run -d -p 8086:8086 \
      -v /data/influxdb2/data:/var/lib/influxdb2 \
      -v /data/influxdb2/config:/etc/influxdb2 \
      -e DOCKER_INFLUXDB_INIT_MODE=setup \
        -e DOCKER_INFLUXDB_INIT_USERNAME=my-user \
      -e DOCKER_INFLUXDB_INIT_PASSWORD=my-password \
      -e DOCKER_INFLUXDB_INIT_ORG=org \
      -e DOCKER_INFLUXDB_INIT_BUCKET=bucket \
        --name influxdb2 \
      influxdb:2.0.6
```

#### 4.3.6 HBase 2.x

```bash
docker run -dti --name hbase2 -h hbase2 \
-p 2181:2181 \
-p 8080:8080 -p 8085:8085 -p 9090:9090 -p 9095:9095 \
-p 16000:16000 -p 16010:16010 -p 16020:16020 -p 16030:16030 \
harisekhon/hbase:2.1

docker exec -it hbase2 /bin/bash
hbase shell
```

```sql
create 'test', 'c1'
describe 'test'
put 'test','10010','c1:name','zhangsan'
get "test", "10010"
put 'test','10010','c1:sex','man'
scan "test", {FORMATTER => 'toString'} 
count 'test'
truncate 'test'
delete "test", "10010", "c1:sex"
deleteall "test", "10010"
```

#### 4.3.7 TDengine

```bash
docker run -d -p 6041:6041 \
    -v /data/taos/conf:/etc/taos \
    -v /data/taos/data:/var/lib/taos \
    -v /data/taos/logs:/var/log/taos \
    --name tdengine 
    tdengine:2.0.19.1
```

### 4.4 数仓工具

#### 4.4.1 ClickHouse

```bash
docker run -d --name clickhouse-server --privileged=true \
  --ulimit nofile=262144:262144 \
  -p 9000:9000 -p 8123:8123 -p 9009:9009 \
  -v /data/clickhouse/log:/var/log/clickhouse \
  -v /data/clickhouse/data:/var/lib/clickhouse \
  yandex/clickhouse-server:21.3.20.1
```

客户端链接

```bash
docker exec -it clickhouse-server /bin/bash
clickhouse-client -m
```

### 4.5 分布式文件系统

#### 4.5.1 FastDFS

```bash
docker run -dti --network=host --name tracker -v /data/fdfs/tracker:/var/fdfs delron/fastdfs tracker 
docker run -dti --network=host --name storage -p 8888:8888 -p 23000:23000  \
   -e TRACKER_SERVER=172.17.17.200:22122 -v /data/fdfs/storage:/var/fdfs delron/fastdfs storage
```

```bash
docker run --net=host --name=fastdfs -e IP=172.17.17.200 -e WEB_PORT=80 -v /data/fdfs:/var/local/fdfs \
    -d registry.cn-beijing.aliyuncs.com/tianzuo/fastdfs
```

```bash
docker exec -it fastdfs /bin/bash
echo "Hello FastDFS!">index.html
fdfs_test /etc/fdfs/client.conf upload index.html
```

#### 4.5.2 MinIO

```bash
docker pull minio/minio:RELEASE.2022-01-04T07-41-07Z

docker run -dit -p 9000:9000 -p 9001:9001 --name minio \
  -v /data/minio/data:/data \
  -v /data/minio/config:/root/.minio \
  minio/minio:RELEASE.2022-01-04T07-41-07Z server /data --console-address ":9001"
```

访问地址：http://192.168.2.100:9001/ ，账户密码：minioadmin/minioadmin

#### 4.5.3 Hadoop 2.x

```bash
docker run -dit --name hadoop2 -h hadoop2 \
 -p 9000:9000 -p 8020:8020 -p 50070:50070 -p 50470:50470 \
 -p 50010:50010 -p 50075:50075 -p 50475:50475 \
 -p 8088:8088 \
 -p 8040:8040 -p 8041:8041 -p 8042:8042 \
 sequenceiq/hadoop-docker:2.7.1 /etc/bootstrap.sh -bash
```

```bash
docker exec -it hadoop2 /bin/bash
# 配置环境
vi /etc/profile
export HADOOP_HOME="/usr/local/hadoop-2.7.1"
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

source /etc/profile
# 统计高频词出现次数
hadoop fs -mkdir -p /wordcount/input
hadoop fs -put a.txt /wordcount/input # 创建a.txt并编辑内容
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar wordcount /wordcount/input /wordcount/output
hadoop fs -cat /wordcount/output/part-r-00000
```

```lua
组件    节点    默认端口    配置    用途说明
HDFS    DataNode    50010    dfs.datanode.address    datanode服务端口，用于数据传输
HDFS    DataNode    50075    dfs.datanode.http.address    http服务的端口
HDFS    DataNode    50475    dfs.datanode.https.address    https服务的端口
HDFS    DataNode    50020    dfs.datanode.ipc.address    ipc服务的端口
HDFS    NameNode    50070    dfs.namenode.http-address    http服务的端口
HDFS    NameNode    50470    dfs.namenode.https-address    https服务的端口
HDFS    NameNode    8020    fs.defaultFS    接收Client连接的RPC端口，用于获取文件系统metadata信息。
HDFS    journalnode    8485    dfs.journalnode.rpc-address    RPC服务
HDFS    journalnode    8480    dfs.journalnode.http-address    HTTP服务
HDFS    ZKFC    8019    dfs.ha.zkfc.port    ZooKeeper FailoverController，用于NN HA
YARN    ResourceManager    8032    yarn.resourcemanager.address    RM的applications manager(ASM)端口
YARN    ResourceManager    8030    yarn.resourcemanager.scheduler.address    scheduler组件的IPC端口
YARN    ResourceManager    8031    yarn.resourcemanager.resource-tracker.address    IPC
YARN    ResourceManager    8033    yarn.resourcemanager.admin.address    IPC
YARN    ResourceManager    8088    yarn.resourcemanager.webapp.address    http服务端口
YARN    NodeManager    8040    yarn.nodemanager.localizer.address    localizer IPC
YARN    NodeManager    8042    yarn.nodemanager.webapp.address    http服务端口
YARN    NodeManager    8041    yarn.nodemanager.address    NM中container manager的端口
YARN    JobHistory Server    10020    mapreduce.jobhistory.address    IPC
YARN    JobHistory Server    19888    mapreduce.jobhistory.webapp.address    http服务端口
HBase    Master    60000    hbase.master.port    IPC
HBase    Master    60010    hbase.master.info.port    http服务端口
HBase    RegionServer    60020    hbase.regionserver.port    IPC
HBase    RegionServer    60030    hbase.regionserver.info.port    http服务端口
HBase    HQuorumPeer    2181    hbase.zookeeper.property.clientPort    HBase-managed ZK mode，使用独立的ZooKeeper集群则不会启用该端口。
HBase    HQuorumPeer    2888    hbase.zookeeper.peerport    HBase-managed ZK mode，使用独立的ZooKeeper集群则不会启用该端口。
HBase    HQuorumPeer    3888    hbase.zookeeper.leaderport    HBase-managed ZK mode，使用独立的ZooKeeper集群则不会启用该端口。
Hive    Metastore    9083    /etc/default/hive-metastore中export PORT=<port>来更新默认端口     
Hive    HiveServer    10000    /etc/hive/conf/hive-env.sh中export HIVE_SERVER2_THRIFT_PORT=<port>来更新默认端口     
ZooKeeper    Server    2181    /etc/zookeeper/conf/zoo.cfg中clientPort=<port>    对客户端提供服务的端口
ZooKeeper    Server    2888    /etc/zookeeper/conf/zoo.cfg中server.x=[hostname]:nnnnn[:nnnnn]，标蓝部分    follower用来连接到leader，只在leader上监听该端口。
ZooKeeper    Server    3888    /etc/zookeeper/conf/zoo.cfg中server.x=[hostname]:nnnnn[:nnnnn]，标蓝部分    用于leader选举的。只在electionAlg是1,2或3(默认)时需要。
```


### 4.6 分布式中间件

#### 4.6.1 Nacos 2.0.1

```bash
docker run --name nacos -e MODE=standalone -p 8848:8848 -d nacos/nacos-server:2.0.1

docker run --name nacos -d --network=host -p 8848:8848 -e MODE=standalone \
-e JVM_XMS=256m -e JVM_XMX=256m \
-e SPRING_DATASOURCE_PLATFORM=mysql \
-e MYSQL_SERVICE_HOST=172.17.17.137 \
-e MYSQL_SERVICE_PORT=3306 \
-e MYSQL_SERVICE_USER=root \
-e MYSQL_SERVICE_PASSWORD=root \
-e MYSQL_SERVICE_DB_NAME=nacos_config \
-e TIME_ZONE='Asia/Shanghai' \
--restart=always nacos/nacos-server:2.0.1
```

#### 4.6.2 Consul 1.12.1

```
docker run -d -p 8500:8500 --restart=always --name=consul consul:1.12.1 agent -server -bootstrap -ui -node=1 -client='0.0.0.0'
```


#### 4.6.3 Seata 1.4.2 

1. 下载镜像

```bash
docker pull seataio/seata-server:1.4.2
```

2. 拷贝文件到宿主机

```bash
docker run -d --name seata-server -p 8091:8091 seataio/seata-server:1.4.2
mkdir -p /data/seata/resources
docker cp seata-server:/seata-server/resources  /data/seata/
docker rm -f seata-server
```

3. 修改配置

```bash
cd /data/seata/resources
```

修改注册信息

```bash
vi registry.conf
```

```conf
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    application = "seata-server"
    serverAddr = "172.17.17.165:8848"
    group = "SEATA_GROUP"
    namespace = "1207ed15-5658-48db-a35a-8e3725930070"
    cluster = "default"
    username = "nacos"
    password = "nacos"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"

  file {
    name = "file.conf"
  }
}
```

修改存储信息

```bash
vi file.conf
```

```conf
store {
  mode = "db"
  db {
    datasource = "druid"
    dbType = "postgresql"
    driverClassName = "org.postgresql.Driver"
    url = "jdbc:postgresql://127.0.0.1:5432/seata"
    user = "postgres"
    password = "postgres"
    minConn = 5
    maxConn = 100
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }
}
```

4. 启动容器

```bash
docker run -dit --name seata-server --restart always \
-p 7091:7091 -p 8091:8091 \
-e SEATA_IP=192.168.2.201 \
-e SEATA_PORT=8091 \
-v /mydata/seata/resources:/seata-server/resources \
seataio/seata-server:1.4.2
```

```bash
SEATA_IP        # 可选, 指定seata-server启动的IP, 该IP用于向注册中心注册时使用, 如eureka等
SEATA_PORT      # 可选, 指定seata-server启动的端口, 默认为 8091
STORE_MODE      # 可选, 指定seata-server的事务日志存储方式, 支持db ,file,redis(Seata-Server 1.3及以上版本支持), 默认是 file
SERVER_NODE     # 可选, 用于指定seata-server节点ID, 如 1,2,3..., 默认为 根据ip生成
SEATA_ENV       # 可选, 指定 seata-server 运行环境, 如 dev, test 等, 服务启动时会使用 registry-dev.conf 这样的配置
SEATA_CONFIG_NAME   # 可选, 指定配置文件位置, 如 file:/root/registry, 将会加载 /root/registry.conf 作为配置文件，
# 如果需要同时指定 file.conf文件，需要将registry.conf的config.file.name的值改为类似file:/root/file.conf：
```

#### 4.6.4 Sentinel 1.7.2

```bash
docker run --name sentinel -d -p 8858:8858 -d bladex/sentinel-dashboard:1.7.2
```

#### 4.6.5 Dubbo Admin

```bash
docker run -d -p 7001:7001 -e dubbo.registry.address=zookeeper://172.17.17.200:2181 \
-e dubbo.admin.root.password=root \
-e dubbo.admin.guest.password=guest \
chenchuxin/dubbo-admin
```

#### 4.6.6 Zookeeper 3.7.0

```bash
docker run -d -p 2181:2181 --name some-zookeeper --restart always -d zookeeper:3.7.0
```

```bash
docker pull zookeeper:3.7.0

mkdir /data/zookeeper/conf/ -p
cd /data/zookeeper/conf/
touch zoo.cfg

# 设置心跳时间，单位毫秒
tickTime=2000
# 存储内存数据库快照的文件夹
dataDir=/tmp/zookeeper
# 监听客户端连接的端口
clientPort=2181

docker run -p 2181:2181 --name zookeeper \
-v /data/zookeeper/conf/zoo.cfg:/conf/zoo.cfg \
-d zookeeper:3.7.0
```


#### 4.6.7 Zipkin 2.23

```bash
docker run -d --name zipkin -p  9411:9411 openzipkin/zipkin:2.23
```

#### 4.6.8 SkyWalking 8.9.1

1. 相关组件版本

| **组件** | **版本** | **端口** |
| :---------- | :----------: | :----------: |
| skywalking-oap-server | 8.9.1 | 11800，12800 |
| skywalking-ui | 8.9.1 | 8080 |
| elasticsearch | 7.17.6 | 9200 |
| apache-skywalking-java-agent| 8.15.0 | xxx |

2. 创建目录赋予权限

```bash
mkdir /data/elasticsearch/{data,logs} -p
chmod 775 /data/elasticsearch/
```

3. 准备docker-compose.yml文件

```yml
version: '3.3'
services:
  elasticsearch:
    image: elasticsearch:7.17.6
    container_name: elasticsearch
    restart: always
    ports:
      - 9200:9200
    environment:
      - "TAKE_FILE_OWNERSHIP=true" #volumes 挂载权限 如果不想要挂载es文件改配置可以删除
      - "discovery.type=single-node" #单机模式启动
      - "TZ=Asia/Shanghai" # 设置时区
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" # 设置jvm内存大小
    volumes:
      - /data/elasticsearch/logs:/usr/share/elasticsearch/logs
      - /data/elasticsearch/data:/usr/share/elasticsearch/data
     #- /data/elasticsearch/conf/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    ulimits:
      memlock:
        soft: -1
        hard: -1
  skywalking-oap-server:
    image: apache/skywalking-oap-server:8.9.1
    container_name: skywalking-oap-server
    depends_on:
      - elasticsearch
    links:
      - elasticsearch
    restart: always
    ports:
      - 11800:11800
      - 12800:12800
    environment:
      SW_STORAGE: elasticsearch  # 指定ES版本
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
      TZ: Asia/Shanghai
    #volumes:
     #- /data/oap/conf/alarm-settings.yml:/skywalking/config/alarm-settings.yml
  skywalking-ui:
    image: apache/skywalking-ui:8.9.1
    container_name: skywalking-ui
    depends_on:
      - skywalking-oap-server
    links:
      - skywalking-oap-server
    restart: always
    ports:
      - 8080:8080
    environment:
      SW_OAP_ADDRESS: http://skywalking-oap-server:12800
      TZ: Asia/Shanghai
```

4. 执行启动服务命令

```bash
docker-compose up -d
```

5. 下载代理

https://archive.apache.org/dist/skywalking/java-agent/8.16.0/apache-skywalking-java-agent-8.16.0.tgz

6. 访问地址

http://192.168.2.201:8080

### 4.7 消息中间件

#### 4.7.1 ActiveMQ 

```bash
docker run -d --name activemq -p 8161:8161 -p 1883:1883 -p 61614:61614 -p 61616:61616  webcenter/activemq:5.14.3
```

控制台地址：http://0.0.0.0:8161
默认账号密码admin/admin

#### 4.7.2 RabbitMQ

```bash
docker run -p 5672:5672 -p 15672:15672 --name rabbitmq -d rabbitmq:3.7.15
# 下载插件
wget https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/v3.8.0/rabbitmq_delayed_message_exchange-3.8.0.ez
docker cp rabbitmq_delayed_message_exchange-3.8.0.ez rabbitmq:/plugins/
docker exec -it rabbitmq /bin/bash
# 启动插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
rabbitmq-plugins enable rabbitmq_management
rabbitmqctl add_user admin 123456
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"
```

控制台地址：http://0.0.0.0:15672
账号密码admin/123456

```bash
# 其他版本
docker run -dit --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.9-management
```

#### 4.7.3 RocketMQ

1. 创建目录

```bash
mkdir -p /data/rocketmq/data/namesrv/logs /root/rocketmq/data/namesrv/store /data/rocketmq/conf /data/rocketmq/data/broker/logs /data/rocketmq/data/broker/stor
```

2. 创建broker.conf配置

```bash
cd /data/rocketmq/conf
vi broker.conf
```

```conf
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
brokerIP1 = 172.17.17.200
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

3. 拉取镜像

```bash
docker pull rocketmqinc/rocketmq:4.4.0
docker pull apacherocketmq/rocketmq-dashboard:1.0.0
```

4. 启动namesrv

```bash
docker run -d -p 9876:9876 -v /data/rocketmq/data/namesrv/logs:/root/logs -v /data/rocketmq/data/namesrv/store:/root/store --name rmqnamesrv -e "MAX_POSSIBLE_HEAP=100000000" rocketmqinc/rocketmq:4.4.0 sh mqnamesrv
```

5. 启动broker

```
docker run -d -p 10911:10911 -p 10909:10909 -v  /data/rocketmq/data/broker/logs:/root/logs -v  /data/rocketmq/data/broker/store:/root/store -v  /data/rocketmq/conf/broker.conf:/opt/rocketmq-4.4.0/conf/broker.conf --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" -e "MAX_POSSIBLE_HEAP=200000000" rocketmqinc/rocketmq:4.4.0 sh mqbroker -c /opt/rocketmq-4.4.0/conf/broker.conf
```

6. 启动dashboard

```bash
docker run -d --name rocketmq-dashboard -e "JAVA_OPTS=-Drocketmq.namesrv.addr=172.17.17.200:9876" -p 9080:8080 -t apacherocketmq/rocketmq-dashboard:1.0.0
```

#### 4.7.4 Kafka

```
docker pull wurstmeister/kafka:2.13-2.8.1
```

```bash
docker run -itd --name kafka -p 9092:9092 \
  --link some-zookeeper:zookeeper \
  -e KAFKA_BROKER_ID=0 \
  -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.17.17.200:9092 \
  -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
  -t wurstmeister/kafka:2.13-2.8.1
```

```bash
docker run -itd --name kafka-manager -p 9000:9000 -e ZK_HOSTS="172.17.17.200:2181" -e APPLICATION_SECRET=letmein sheepkiller/kafka-manager
```

#### 4.7.5 Pulsar


```bash
docker pull apachepulsar/pulsar:2.8.4
docker run -dit \
    -p 6650:6650 \
    -p 8080:8080 \
    -v pulsardata:/data/pulsar/data \
    -v pulsarconf:/data/pulsar/conf \
    --name pulsar-standalone \
    apachepulsar/pulsar:2.8.4 \
    bin/pulsar standalone
```

```bash
docker pull apachepulsar/pulsar-manager:v0.2.0
docker run -dit \
    -p 9527:9527 -p 7750:7750 \
    -e SPRING_CONFIGURATION_FILE=/pulsar-manager/pulsar-manager/application.properties \
    --link pulsar-standalone \
    apachepulsar/pulsar-manager:v0.2.0
```

添加管理员

```bash
CSRF_TOKEN=$(curl http://localhost:7750/pulsar-manager/csrf-token)

curl \
    -H "X-XSRF-TOKEN: $CSRF_TOKEN" \
    -H "Cookie: XSRF-TOKEN=$CSRF_TOKEN;" \
    -H 'Content-Type: application/json' \
    -X PUT http://localhost:7750/pulsar-manager/users/superuser \
    -d '{"name": "admin", "password": "12345678", "description": "admin", "email": "username@test.org"}'
```

访问地址：http://192.168.2.201:9527
账号密码admin/12345678

添加Environment连接集群：http://192.168.2.201:8080


#### 4.7.6 EMQX

```bash
docker run -d --name emqx -p 1883:1883 -p 8081:8081 -p 8083:8083 -p 8084:8084 -p 8883:8883 -p 18083:18083 emqx/emqx:4.3.10        # 开源版
docker run -d --name emqx-ee -p 1883:1883 -p 8081:8081 -p 8083:8083 -p 8084:8084 -p 8883:8883 -p 18083:18083 emqx/emqx-ee:4.2.9   # 企业版
```



### 4.8 Elastic Stack

#### 4.8.1 Elasticsearch

1. 拉取镜像

```bash
docker pull elasticsearch:7.6.2
```

2. 临时修改虚拟内存区域大小，否则会因为过小而无法启动

```bash
sysctl -w vm.max_map_count=262144
```

3. 目录授权

```bash
chmod 777 /data/elasticsearch/data/
```

4. 启动服务

```bash
docker run -p 9200:9200 -p 9300:9300 --name elasticsearch \
-e "discovery.type=single-node" \
-e "cluster.name=elasticsearch" \
-v /data/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-v /data/elasticsearch/data:/usr/share/elasticsearch/data \
-d elasticsearch:7.6.2
```

5. 安装中文分词器IKAnalyzer

```bash
docker exec -it elasticsearch /bin/bash
#此命令需要在容器中运行
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip
docker restart elasticsearch
http://192.168.3.200:9200/_cat/plugins
```

6. 安装elasticsearch-head插件

```bash
docker run -d -p 9100:9100 docker.io/mobz/elasticsearch-head:5
```

elasticsearch.yml，在文件末尾加入以下配置开启跨域

```yml
http.cors.enabled: true
http.cors.allow-origin: "*"
```

#### 4.8.2 Logstash

1. 拉取镜像

```bash
docker pull logstash:7.6.2
```

2. 创建logstash.conf

```
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
    hosts => "es:9200"
    index => "mall-%{type}-%{+YYYY.MM.dd}"
  }
}
```

3. 创建`/data/logstash`目录，并将Logstash的配置文件`logstash.conf`拷贝到该目录；

```bash
mkdir /data/logstash
```

4. 使用如下命令启动Logstash服务；

```bash
docker run --name logstash -p 4560:4560 -p 4561:4561 -p 4562:4562 -p 4563:4563 \
--link elasticsearch:es \
-v /data/logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
-d logstash:7.6.2
```

5. 进入容器内部，安装`json_lines`插件。

```bash
docker exec -it logstash /bin/bash
cd /usr/share/logstash/bin
logstash-plugin install logstash-codec-json_lines
```

#### 4.8.3 Kibana

访问地址：http://192.168.3.200:5601

```bash
docker pull kibana:7.6.2

docker run --name kibana -p 5601:5601 \
--link elasticsearch:es \
-e "elasticsearch.hosts=http://es:9200" \
-d kibana:7.6.2
```

### 4.9 持续集成

#### 4.9.1 GitLab

拉取镜像

```bash
docker pull gitlab/gitlab-ce:12.4.2-ce.0
mkdir -p /data/gitlab/{config,logs,data}  # 创建目录
```

启动服务

```bash
docker run -d  \
-p 443:443 \
-p 80:80 \
-p 22:22 \
--name gitlab \
--restart always \
-v /data/gitlab/config:/etc/gitlab \
-v /data/gitlab/logs:/var/log/gitlab \
-v /data/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce:12.4.2-ce.0
```

修改配置
```bash
# 修改配置，gitlab.rb文件内容默认全是注释
vi /data/gitlab/config/gitlab.rb
# 配置ssh协议所使用的访问地址和端口
gitlab_rails['gitlab_ssh_host'] = '192.168.2.201'
gitlab_rails['gitlab_shell_ssh_port'] = 22
```

重启服务
```bash
docker restart gitlab
```


#### 4.9.2 Nexus3

```bash
docker pull sonatype/nexus3:3.36.0
mkdir -p /home/mvn/nexus-data  && chown -R 200 /home/mvn/nexus-data
docker run -d -p 8081:8081 --name nexus -v /home/mvn/nexus-data:/nexus-data sonatype/nexus3:3.36.0
```

#### 4.9.3 Harbor

1. 上传解压

```shell
mkdir /opt/softwarte
wget https://github.com/goharbor/harbor/releases/download/v2.0.1/harbor-offline-installer-v2.0.1.tgz
tar xf harbor-offline-installer-v2.0.1.tgz -C /opt/
```


2. 修改配置

```bash
cd /opt/harbor
cp harbor.yml.tmpl harbor.yml
vi harbor.yml
```

```yml
#修改配置文件(如果不用https就注释掉https的几项，我是用的http就注释掉了https的，其他几项修改为自己的信息)
···
hostname: 192.168.3.200
···
# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 88
···
# https related config
#https:
  # https port for harbor, default is 443
  #port: 443
  # The path of cert and key files for nginx
  #certificate: /your/certificate/path
  #private_key: /your/private/key/path
····
harbor_admin_password: Harbor12345
···
data_volume: /data
···
```

3. 安装

```bash
./install.sh 
docker-compose up -d    # 启动
docker-compose stop     # 停止
docker-compose restart  # 重新启动
```

默认账户密码：admin/Harbor12345


#### 4.9.4 SonarQube

```bash
docker run -d --name sonarqube -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true -p 9000:9000 sonarqube:8.6-community #H2默认存储

#postgres外挂存储
docker run -d --name sonarqube \
    --link postgres2 \
    -p 9000:9000 \
    -e sonar.jdbc.url=jdbc:postgresql://172.17.17.80:5432/sonardb \
    -e sonar.jdbc.username=sonar \
    -e sonar.jdbc.password=123456 \
    -v /data/sonarqube/sonarqube_extensions:/opt/sonarqube/extensions \
    -v /data/sonarqube/sonarqube_logs:/opt/sonarqube/logs \
    -v /data/sonarqube/sonarqube_data:/opt/sonarqube/data \
    sonarqube:8.6-community
```

#### 4.9.5 Jenkins

```bash
docker pull jenkins/jenkins:lts
docker pull jenkins/jenkins

mkdir -p /data/jenkins_home/
chown -R 1000:1000 /data/jenkins_home/

docker run -d --name jenkins -p 8888:8080 -p 50000:50000 -v /data/jenkins_home:/var/jenkins_home jenkins/jenkins

cd /data/jenkins_home
vi hudson.model.UpdateCenter.xml
# http://mirror.xmission.com/jenkins/updates/update-center.json
```

#### 4.9.6 Prometheus

```bash
docker pull prom/prometheus
docker pull prom/node-exporter
docker pull grafana/grafana
```

docker-compose-prometheus.yml
```yml
version: '2'
services:
    # prometheus
  prometheus:
    image: "prom/prometheus"
    hostname: prometheus
    container_name: prometheus
    ports:
      - '9090:9090'
    volumes:
      - /data/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    restart: always

    # node-exporter
  node-exporter:
    image: "prom/node-exporter"
    hostname: node-exporter
    container_name: node-exporter
    ports:
      - '9100:9100'
    volumes:
      - /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    restart: always
    network_mode: host
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
```

prometheus.yml
```yml
global:
  scrape_interval:     60s
  evaluation_interval: 60s
 
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus
 
  - job_name: linux
    static_configs:
      - targets: ['172.17.17.201:9100']
```

```bash
docker-compose -f docker-compose-prometheus.yml up -d  
```

#### 4.9.7 Grafana

```bash
docker run -d -p 3000:3000 --name grafana grafana/grafana
```

admin:admin


### 4.10 即时通讯

#### 4.10.1 SRS

```bash
docker pull ossrs/srs:v4.0.187
# 默认安装
docker run --rm -p 1935:1935 -p 1985:1985 -p 8080:8080 \
    -v /home/srs-4.0.187/trunk/conf/srs.conf:/usr/local/srs/conf/srs.conf \
    ossrs/srs:v4.0.187
```

```bash
# 创建自定义网络
docker network create --driver bridge --subnet 172.0.0.0/16 xzh_network
# 查看已存在网络
docker network ls
# 创建数据目录
mkdir -p /home/docker/srs4

# 安装并启动 srs
docker run -p 1935:1935 -p 1985:1985 -p 8080:8080 \
--name srs \
ossrs/srs:v4.0.187
# 把容器中的配置文件复制出来
docker cp -a srs:/usr/local/srs/conf /home/docker/srs4/conf
# 把容器中的数据文件复制出来
docker cp -a srs:/usr/local/srs/objs /home/docker/srs4/objs
# 删除 srs 容器
docker rm -f srs

# 本地配置文件启动
docker run -p 1935:1935 -p 1985:1985 -p 8080:8080 \
--name srs \
--network xzh_network \
--ip 172.0.0.35 \
--restart=always \
-v /home/docker/srs4/conf/:/usr/local/srs/conf/ \
-v /home/docker/srs4/objs/:/usr/local/srs/objs/ \
ossrs/srs:v4.0.187 
```

http://服务器IP地址:8080

SRS自定义参考配置

```conf
listen              1935;
max_connections     1000;
srs_log_tank        file;
srs_log_file        ./objs/srs.log;
daemon              on;
http_api {
    enabled         on;
    listen          1985;
}
http_server {
    enabled         on;
    listen          8080;
    dir             ./objs/nginx/html;
    # 开启 https 支持，需要开放 8088端口
    # https {
        # enabled on;
        # listen 8088;
        # key ./conf/woniu.key;
        # cert ./conf/woniu.crt;
    # }
}
vhost __defaultVhost__ {
    # http-flv设置
    http_remux{
        enabled    on;
        mount      [vhost]/[app]/[stream].flv;
        hstrs      on;
    }
 
    # hls设置
    hls {
        enabled         on;
        hls_fragment    1;
        hls_window      2;
        hls_path        ./objs/nginx/html;
        hls_m3u8_file   [app]/[stream].m3u8;
        hls_ts_file     [app]/[stream]-[seq].ts;
    }

    # dvr设置
    dvr {
        enabled             off;
        dvr_path            ./objs/nginx/html/[app]/[stream]/[2006]/[01]/[02]/[timestamp].flv;
        dvr_plan            segment;
        dvr_duration        30;
        dvr_wait_keyframe   on;
    }

    # rtc 设置
    rtc {
        enabled     on;
        bframe      discard;
    }

    # SRS支持refer防盗链：检查用户从哪个网站过来的。譬如不是从公司的页面过来的人都不让看。
    refer {
        # whether enable the refer hotlink-denial.
        # default: off.
        enabled         off;
        # the common refer for play and publish.
        # if the page url of client not in the refer, access denied.
        # if not specified this field, allow all.
        # default: not specified.
        all           github.com github.io;
        # refer for publish clients specified.
        # the common refer is not overrided by this.
        # if not specified this field, allow all.
        # default: not specified.
        publish   github.com github.io;
        # refer for play clients specified.
        # the common refer is not overrided by this.
        # if not specified this field, allow all.
        # default: not specified.
        play      github.com github.io;
    }
    
    # http 回调
    http_hooks {
    
        # 事件：发生该事件时，即回调指定的HTTP地址。
        # HTTP地址：可以支持多个，以空格分隔，SRS会依次回调这些接口。
        # 数据：SRS将数据POST到HTTP接口。
        # 返回值：SRS要求HTTP服务器返回HTTP200并且response内容为整数错误码（0表示成功），其他错误码会断开客户端连接。
        
        # whether the http hooks enable.
        # default off.
        enabled         on;
        
        # 当客户端连接到指定的vhost和app时
        on_connect      http://127.0.0.1:8085/api/v1/clients http://localhost:8085/api/v1/clients;
        
        # 当客户端关闭连接，或者SRS主动关闭连接时
        on_close        http://127.0.0.1:8085/api/v1/clients http://localhost:8085/api/v1/clients;
       
        # 当客户端发布流时，譬如flash/FMLE方式推流到服务器
        on_publish      http://127.0.0.1:8085/api/v1/streams http://localhost:8085/api/v1/streams;
        
        # 当客户端停止发布流时
        on_unpublish    http://127.0.0.1:8085/api/v1/streams http://localhost:8085/api/v1/streams;
        
        # 当客户端开始播放流时
        on_play         http://127.0.0.1:8085/api/v1/sessions http://localhost:8085/api/v1/sessions;
        
        # 当客户端停止播放时。备注：停止播放可能不会关闭连接，还能再继续播放。
        on_stop         http://127.0.0.1:8085/api/v1/sessions http://localhost:8085/api/v1/sessions;
        
        # 当DVR录制关闭一个flv文件时
        on_dvr          http://127.0.0.1:8085/api/v1/dvrs http://localhost:8085/api/v1/dvrs;
        
        # 当HLS生成一个ts文件时
        on_hls          http://127.0.0.1:8085/api/v1/hls http://localhost:8085/api/v1/hls;
        
        # when srs reap a ts file of hls, call this hook,
        on_hls_notify   http://127.0.0.1:8085/api/v1/hls/[app]/[stream]/[ts_url][param];
    }
}


```

指定配置文件启动 srs
```bash
./objs/srs -c conf/srs.woniu.conf
```

#### 4.10.2 Openfire

-  4.4.4
```bash
docker run --name openfire -d --restart=always \
  --publish 9090:9090 --publish 5222:5222 --publish 7070:7070 \
  --volume /srv/docker/openfire:/var/lib/openfire \
  gizmotronic/openfire:4.4.4
```

### 4.11 其他

#### 4.11.1 Zentao

```bash
mkdir -p /data/zbox
docker run -d -p 8080:80 -p 3316:3306 -e USER="admin" -e PASSWD="admin" -e BIND_ADDRESS="false" -e SMTP_HOST="163.177.90.125 smtp.exmail.qq.com" -v /data/zbox/:/opt/zbox/ --name zentao-server idoop/zentao:latest 
```

- 8080 访问禅道外部端口号
- 3316 把容器3306数据库端口映射到主机3316端口
- USER 设置登录账号 admin
- PASSWD 设置登录密码 123456
- BIND_ADDRESS 设置为false


#### 4.11.2 Node-RED

```bash
sudo docker run -it -p 1880:1880 --name=nodered --restart=always --user=root --net=host -v /data/nodered:/data -e TZ=Asia/Shanghai nodered/node-red
```


#### 4.11.3 Eclipse Che

http://ip:8080

```bash
docker run -it -d --rm \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /data/che:/data \
eclipse/che start
```

#### 4.11.4 Theia

```bash
chown -R 1000:1000 /data/
docker run -it -d -p 3000:3000 -v "/data/theia:/home/project:cached" theiaide/theia
docker run -it -d -p 3000:3000 -v "/data/theia-java:/home/project:cached" theiaide/theia-java
docker run -it -d --init -p 3000:3000 -v "/data/theia-full:/home/project:cached" theiaide/theia-full
```