# Hadoop 3.1.4

## 1. Hadoop3编译安装

基础环境
```bash
bin/hadoop checknative
mkdir -p /export/server/    # 软件安装路径
mkdir -p /export/data/      # 数据存储路径
mkdir -p /export/software/  # 软件包存放路径
```

### 1.1 安装编译相关的依赖

```bash
yum install gcc gcc-c++
yum install make cmake
yum install autoconf automake libtool curl
yum install lzo-devel zlib-devel openssl openssl-devel ncurses-devel
yum install snappy snappy-devel bzip2 bzip2-devel lzo lzo-devel lzop libXtst
```

### 1.2 手动安装cmake 

```bash
yum erase cmake                 # yum卸载已安装cmake 版本低
tar zxvf cmake-3.13.5.tar.gz    # 解压
cd /export/server/cmake-3.13.5  # 编译安装
./configure
make && make install
cmake -version                  # 验证,如果没有正确显示版本 请断开SSH连接 重写登录
```

### 1.3 手动安装snappy

```bash
#卸载已经安装的
cd /usr/local/lib
rm -rf libsnappy*
#上传解压
tar zxvf snappy-1.1.3.tar.gz 
#编译安装
cd /export/server/snappy-1.1.3
./configure
make && make install
#验证是否安装
[root@node4 snappy-1.1.3]# ls -lh /usr/local/lib |grep snappy
-rw-r--r-- 1 root root 511K Nov  4 17:13 libsnappy.a
-rwxr-xr-x 1 root root  955 Nov  4 17:13 libsnappy.la
lrwxrwxrwx 1 root root   18 Nov  4 17:13 libsnappy.so -> libsnappy.so.1.3.0
lrwxrwxrwx 1 root root   18 Nov  4 17:13 libsnappy.so.1 -> libsnappy.so.1.3.0
-rwxr-xr-x 1 root root 253K Nov  4 17:13 libsnappy.so.1.3.0
```

### 1.4 安装配置JDK1.8

```bash
#解压安装包
tar zxvf jdk-8u65-linux-x64.tar.gz

#配置环境变量
vim /etc/profile

export JAVA_HOME=/export/server/jdk1.8.0_65
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

source /etc/profile

#验证是否安装成功
java -version
```

### 1.5 安装配置maven

```bash
#解压安装包
tar zxvf apache-maven-3.5.4-bin.tar.gz

#配置环境变量
vim /etc/profile

export MAVEN_HOME=/export/server/apache-maven-3.5.4
export MAVEN_OPTS="-Xms4096m -Xmx4096m"
export PATH=:$MAVEN_HOME/bin:$PATH

source /etc/profile

#验证是否安装成功
[root@node4 ~]# mvn -v
Apache Maven 3.5.4

#添加maven 阿里云仓库地址 加快国内编译速度
vim /export/server/apache-maven-3.5.4/conf/settings.xml

<mirrors>
     <mirror>
           <id>alimaven</id>
           <name>aliyun maven</name>
           <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
           <mirrorOf>central</mirrorOf>
      </mirror>
</mirrors>
```

### 1.6 安装ProtocolBuffer 2.5.0

```bash
#解压
tar zxvf protobuf-2.5.0.tar.gz

#编译安装
cd /export/server/protobuf-2.5.0
./configure
make && make install

#验证是否安装成功
protoc --version
```

### 1.7 编译hadoop

```bash
#上传解压源码包
tar zxvf hadoop-3.1.4-src.tar.gz

#编译
cd /export/server/hadoop-3.1.4-src

mvn clean package -Pdist,native -DskipTests -Dtar -Dbundle.snappy -Dsnappy.lib=/usr/local/lib

#参数说明：
Pdist,native    # 把重新编译生成的hadoop动态库；
DskipTests      # 跳过测试
Dtar            # 最后把文件以tar打包
Dbundle.snappy  # 添加snappy压缩支持【默认官网下载的是不支持的】
Dsnappy.lib=/usr/local/lib # 指snappy在编译机器上安装后的库路径
```


## 2. Hadoopn集群安装

节点规划
- node1   namenode datanode resourcemanager nodemanager
- node2   secondarynamenode datanode nodemanager 
- node3   datanode nodemanager

### 2.1.1 配置环境（hadoop-env.sh）

```bash
cd /export/software

tar -xvzf hadoop-3.1.4.tar.gz -C ../server  # 解压
mkdir -p /export/server/hadoop-3.1.4/data/namenode
mkdir -p /export/server/hadoop-3.1.4/data/datanode
mkdir -p /export/data/hadoop-3.1.4
cd /export/server/hadoop-3.1.4/etc/hadoop/
vim hadoop-env.sh
```

```yaml
# 配置JAVA_HOME
export JAVA_HOME=/export/server/jdk1.8.0_65
# 设置用户以执行对应角色shell命令
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root 
```

### 2.1.2 配置NameNode（core-site.xml）

```bash
cd /export/server/hadoop-3.1.4/etc/hadoop/
vim core-site.xml
```
```yaml
<!-- 默认文件系统的名称。通过URI中schema区分不同文件系统。-->
<!-- file:///本地文件系统 hdfs:// hadoop分布式文件系统 gfs://。-->
<!-- hdfs文件系统访问地址：http://nn_host:8020。-->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://node1.itcast.cn:8020</value>
</property>
<!-- hadoop本地数据存储目录 format时自动生成 -->
<property>
    <name>hadoop.tmp.dir</name>
    <value>/export/data/hadoop-3.1.4</value>
</property>
<!-- 在Web UI访问HDFS使用的用户名。-->
<property>
    <name>hadoop.http.staticuser.user</name>
    <value>root</value>
</property>
```

### 2.1.3 配置HDFS路径（hdfs-site.xml）

```bash
cd /export/server/hadoop-3.1.4/etc/hadoop/
vim hdfs-site.xml
```
```yaml
<!-- 设定SNN运行主机和端口。-->
<property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>node2.itcast.cn:9868</value>
</property>
```

### 2.1.4 配置MapReduce（mapred-site.xml）

```bash
cd /export/server/hadoop-3.1.4/etc/hadoop/
vim mapred-site.xml
```
```yaml
<!-- mr程序默认运行方式。yarn集群模式 local本地模式-->
<property>
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>
<!-- MR App Master环境变量。-->
<property>
  <name>yarn.app.mapreduce.am.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
<!-- MR MapTask环境变量。-->
<property>
  <name>mapreduce.map.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
<!-- MR ReduceTask环境变量。-->
<property>
  <name>mapreduce.reduce.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
```

### 2.1.5 配置YARN（yarn-site.xml）

```bash
cd /export/server/hadoop-3.1.4/etc/hadoop/
vim yarn-site.xml
```
```yaml
<!-- yarn集群主角色RM运行机器。-->
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>node1.itcast.cn</value>
</property>
<!-- NodeManager上运行的附属服务。需配置成mapreduce_shuffle,才可运行MR程序。-->
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
<!-- 每个容器请求的最小内存资源（以MB为单位）。-->
<property>
  <name>yarn.scheduler.minimum-allocation-mb</name>
  <value>512</value>
</property>
<!-- 每个容器请求的最大内存资源（以MB为单位）。-->
<property>
  <name>yarn.scheduler.maximum-allocation-mb</name>
  <value>2048</value>
</property>
<!-- 容器虚拟内存与物理内存之间的比率。-->
<property>
  <name>yarn.nodemanager.vmem-pmem-ratio</name>
  <value>4</value>
</property>
```

### 2.1.6 配置从节点（workers）

```bash
cd /export/server/hadoop-3.1.4/etc/hadoop/
vim workers
```

```yaml
# 删除第一行localhost，然后添加以下三行
node1.itcast.cn
node2.itcast.cn
node3.itcast.cn
```

### 2.1.7 分发安装包

```bash
cd /export/server/
scp -r hadoop-3.1.4 root@node2:/export/server/
scp -r hadoop-3.1.4 root@node3:/export/server/
```

### 2.1.8 配置Hadoop环境变量

```bash
vim /etc/profile
export HADOOP_HOME=/export/server/hadoop-3.1.4
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
source /etc/profile

scp /etc/profile root@node2:/etc/
scp /etc/profile root@node3:/etc/
```

### 2.1.9 格式化启动

```bash
hdfs namenode -format itcast-hadoop
```
单节点启动
```bash
# HDFS集群
hdfs --daemon start namenode|datanode|secondarynamenode
hdfs --daemon stop  namenode|datanode|secondarynamenode
# YARN集群
yarn --daemon start resourcemanager|nodemanager
yarn --daemon stop  resourcemanager|nodemanager
```
一键启动
```bash
# HDFS集群
start-dfs.sh 
stop-dfs.sh 
# YARN集群
start-yarn.sh
stop-yarn.sh
# Hadoop集群
start-all.sh
stop-all.sh 
```

### 2.1.10 Web UI

HDFS集群 http://namenode_host:9870

YARN集群 http://resourcemanager_host:8088



## 3. Hive

Apache Hive是一款建立在Hadoop之上的开源数据仓库系统，可以将存储在Hadoop文件中的结构化、半结构化数据文件映射为一张数据库表，基于表提供了一种类似SQL的查询模型，称为Hive查询语言（HQL），用于访问和分析存储在Hadoop文件中的大型数据集。
Hive核心是将HQL转换为MapReduce程序，然后将程序提交到Hadoop群集执行。Hive由Facebook实现并开源

Hive不是分布式安装运行的软件，其分布式的特性主要借由Hadoop完成。包括分布式存储、分布式计算

hive 3.1.2对应hadoop3.1.4

### 3.1 安装部署

#### 3.1.1 内嵌模式
```bash
#--------------------Hive安装配置----------------------
# 上传解压安装包
cd /export/server/
tar zxvf apache-hive-3.1.2-bin.tar.gz
mv apache-hive-3.1.2-bin hive

#解决hadoop、hive之间guava版本差异
cd /export/server/hive
rm -rf lib/guava-19.0.jar
cp /export/server/hadoop-3.1.4/share/hadoop/common/lib/guava-27.0-jre.jar ./lib/

#修改hive环境变量文件 添加HADOOP_HOME
cd /export/server/hive/conf/
mv hive-env.sh.template hive-env.sh
vim hive-env.sh
export HADOOP_HOME=/export/server/hadoop-3.1.4
export HIVE_CONF_DIR=/export/server/hive/conf
export HIVE_AUX_JARS_PATH=/export/server/hive/lib

#初始化metadata
cd /export/server/hive
bin/schematool -dbType derby -initSchema

#启动hive服务
bin/hive
```

#### 3.1.2 本地模式
```bash
#--------------------Hive安装配置----------------------
# 上传解压安装包
cd /export/server/
tar zxvf apache-hive-3.1.2-bin.tar.gz
mv apache-hive-3.1.2-bin hive

#解决hadoop、hive之间guava版本差异
cd /export/server/hive
rm -rf lib/guava-19.0.jar
cp /export/server/hadoop-3.1.4/share/hadoop/common/lib/guava-27.0-jre.jar ./lib/

#添加mysql jdbc驱动到hive安装包lib/文件下
mysql-connector-java-5.1.32.jar

#修改hive环境变量文件 添加HADOOP_HOME
cd /export/server/hive/conf/
mv hive-env.sh.template hive-env.sh
vim hive-env.sh
export HADOOP_HOME=/export/server/hadoop-3.1.4
export HIVE_CONF_DIR=/export/server/hive/conf
export HIVE_AUX_JARS_PATH=/export/server/hive/lib

#新增hive-site.xml 配置mysql等相关信息
vim hive-site.xml


#初始化metadata
cd /export/server/hive
bin/schematool -initSchema -dbType mysql -verbos
#初始化成功会在mysql中创建74张表

#启动hive服务
bin/hive

#-----------------Hive-site.xml---------------------
<configuration>
    <!-- 存储元数据mysql相关配置 -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value> jdbc:mysql://node1:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hadoop</value>
    </property>

    <!-- 关闭元数据存储授权  -->
    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>

    <!-- 关闭元数据存储版本的验证 -->
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
</configuration>
```


#### 3.1.3 远程模式

远程模式比本地模式需要单独启动metastore服务
```bash
#--------------------Hive安装配置----------------------
# 上传解压安装包
cd /export/server/
tar zxvf apache-hive-3.1.2-bin.tar.gz
mv apache-hive-3.1.2-bin hive

#解决hadoop、hive之间guava版本差异
cd /export/server/hive
rm -rf lib/guava-19.0.jar
cp /export/server/hadoop-3.1.4/share/hadoop/common/lib/guava-27.0-jre.jar ./lib/

#添加mysql jdbc驱动到hive安装包lib/文件下
mysql-connector-java-5.1.32.jar

#修改hive环境变量文件 添加HADOOP_HOME
cd /export/server/hive/conf/
mv hive-env.sh.template hive-env.sh
vim hive-env.sh
export HADOOP_HOME=/export/server/hadoop-3.1.4
export HIVE_CONF_DIR=/export/server/hive/conf
export HIVE_AUX_JARS_PATH=/export/server/hive/lib

#新增hive-site.xml 配置mysql等相关信息
vim hive-site.xml


#初始化metadata
cd /export/server/hive
bin/schematool -initSchema -dbType mysql -verbos
#初始化成功会在mysql中创建74张表


#-----------------hive-site.xml--------------
<configuration>
    <!-- 存储元数据mysql相关配置 -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value> jdbc:mysql://node1:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hadoop</value>
    </property>

    <!-- H2S运行绑定host -->
    <property>
        <name>hive.server2.thrift.bind.host</name>
        <value>node1</value>
    </property>

    <!-- 远程模式部署metastore 服务地址 -->
    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://node1:9083</value>
    </property>

    <!-- 关闭元数据存储授权  -->
    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>

    <!-- 关闭元数据存储版本的验证 -->
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
</configuration>



#-----------------Metastore Hiveserver2启动----
#前台启动  关闭ctrl+c
/export/server/hive/bin/hive --service metastore

#后台启动 进程挂起  关闭使用jps + kill
#输入命令回车执行 再次回车 进程将挂起后台
nohup /export/server/hive/bin/hive --service metastore &

#前台启动开启debug日志
/export/server/hive/bin/hive --service metastore --hiveconf hive.root.logger=DEBUG,console
```