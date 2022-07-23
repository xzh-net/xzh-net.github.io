# Hadoop 3.1.4

## 1.编译

### 1.1 安装依赖

```bash
yum install gcc gcc-c++
yum install make cmake  # 这里cmake版本推荐为3.6版本以上，版本低源码无法编译！可手动安装
yum install autoconf automake libtool curl
yum install lzo-devel zlib-devel openssl openssl-devel ncurses-devel
yum install snappy snappy-devel bzip2 bzip2-devel lzo lzo-devel lzop libXtst
```

### 1.2 安装cmake

```bash
yum erase cmake         # yum卸载版本低cmake
cd /opt/software/ 
tar zxvf cmake-3.13.5.tar.gz  -C /opt
cd /opt/cmake-3.13.5
# 编译安装
./configure
make && make install
cmake -version
```

如果编译失败提示找不到`GLIBCXX_3.4.21`，就升级gcc9以后，将libstdc++.so.6复制到提示路径下替换旧文件即可

```bash
find / -name "libstdc++.so*"
strings /opt/gcc9/lib64/libstdc++.so.6|grep GLIBCXX
cp -r /opt/gcc9/lib64/libstdc++.so.6 /lib64/
```

### 1.3 安装snappy

```bash 
cd /usr/local/lib   
rm -rf libsnappy*   # 卸载已经安装的
# 上传解压
cd /opt/software/ 
tar zxvf snappy-1.1.3.tar.gz -C /opt
cd /opt/snappy-1.1.3
# 编译安装
./configure
make && make install
# 验证是否安装
ls -lh /usr/local/lib |grep snappy
```

### 1.4 安装jdk

```bash
java -version
```

### 1.5 安装配置maven

```bash
mvn -v
```

### 1.6 安装ProtocolBuffer

```bash
cd /opt/software/ 
tar zxvf protobuf-2.5.0.tar.gz -C /opt
# 编译安装
cd /opt/protobuf-2.5.0
./configure
make && make install

#验证是否安装成功
protoc --version
```

### 1.7 编译hadoop

```bash
cd /opt/software/ 
tar zxvf hadoop-3.1.4-src.tar.gz -C /opt
# 编译
cd /opt/hadoop-3.1.4-src
mvn clean package -Pdist,native -DskipTests -Dtar -Dbundle.snappy -Dsnappy.lib=/usr/local/lib
```

参数说明：

```bash
Pdist,native        # 把重新编译生成的hadoop动态库；
DskipTests          # 跳过测试
Dtar                # 最后把文件以tar打包
Dbundle.snappy      # 添加snappy压缩支持【默认官网下载的是不支持的】
Dsnappy.lib=/usr/local/lib  # 指snappy在编译机器上安装后的库路径
```

编译之后的安装包路径

```bash
/opt/hadoop-3.1.4-src/hadoop-dist/target
```

## 2. 集群搭建


### 2.1 集群角色规划

- node01 namenode datanode resourcemanager nodemanager
- node02 secondarynamenode datanode nodemanager 
- node03 datanode nodemanager

### 2.2 基础环境准备(三台机器)

#### 2.2.1 hosts映射

```bash
hostnamectl set-hostname node01.xuzhihao.net
hostnamectl set-hostname node02.xuzhihao.net
hostnamectl set-hostname node03.xuzhihao.net
```

#### 2.2.2 域名映射

```bash
vim /etc/hosts
# 添加
192.168.123.201 node01 node01.xuzhihao.net
192.168.123.202 node02 node02.xuzhihao.net
192.168.123.203 node03 node03.xuzhihao.net
```

#### 2.2.3 关闭防火墙

```bash
systemctl stop firewalld.service 
systemctl disable firewalld.service
```

#### 2.2.4 免密登录

```bash
ssh-keygen # 3个回车 生成公钥、私钥
# 192.168.123.201 执行
ssh-copy-id node01
ssh-copy-id node02
ssh-copy-id node03
```

#### 2.2.5 集群时间同步

```bash
yum -y install ntpdate
ntpdate ntp4.aliyun.com
```

#### 2.2.6 安装jdk

```bash
java -version
```

### 2.3 上传安装包

#### 2.3.1 解压

```bash
cd /opt/software
tar -zxvf hadoop-3.1.4-bin-snappy-CentOS7.tar.gz -C /opt
```

#### 2.3.2 创建目录

```bash
mkdir -p /opt/software/     # 安装包存放路径，已创建
mkdir -p /opt/              # 软件安装路径，已创建
mkdir -p /opt/hadoop-3.1.4/data/    # 数据存储路径
mkdir -p /opt/hadoop-3.1.4/tmp/     # 临时数据存储路径
```

### 2.4 修改配置

#### 2.4.1 hadoop-env.sh

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim hadoop-env.sh
# 编辑内容
export JAVA_HOME=/usr/local/jdk1.8.0_211
# 添加到末尾，设置用户以执行对应角色shell命令
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root 
```

#### 2.4.2 core-site.xml

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim core-site.xml

# 添加到configuration区间
<!-- 默认文件系统的名称。通过URI中schema区分不同文件系统。-->
<!-- file:///本地文件系统 hdfs:// hadoop分布式文件系统 gfs://。-->
<!-- hdfs文件系统访问地址：http://nn_host:8020。-->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://node01.xuzhihao.net:8020</value>
</property>
<!-- hadoop本地数据存储目录 format时自动生成 -->
<property>
    <name>hadoop.tmp.dir</name>
    <value>/opt/hadoop-3.1.4/tmp</value>
</property>
<!-- 在Web UI访问HDFS使用的用户名。-->
<property>
    <name>hadoop.http.staticuser.user</name>
    <value>root</value>
</property>
```

#### 2.4.3 hdfs-site.xml

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim hdfs-site.xml

# 添加到configuration区间
<!-- 设定SNN运行主机和端口。-->
<property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>node02.xuzhihao.net:9868</value>
</property>
```

#### 2.4.4 mapred-site.xml

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim mapred-site.xml

# 添加到configuration区间
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

#### 2.4.5 yarn-site.xml

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim yarn-site.xml

# 添加到configuration区间
<!-- yarn集群主角色RM运行机器。-->
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>node01.xuzhihao.net</value>
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

#### 2.4.6 配置从角色

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim workers

# 删除第一行localhost，然后添加以下三行
node01.xuzhihao.net
node02.xuzhihao.net
node03.xuzhihao.net
```

### 2.5 分发安装包

```bash
cd /opt/hadoop-3.1.4
scp -r /opt/hadoop-3.1.4 root@node02.xuzhihao.net:$PWD
scp -r /opt/hadoop-3.1.4 root@node03.xuzhihao.net:$PWD
```

### 2.6 设置Hadoop环境变量

```bash
vim /etc/profile
export HADOOP_HOME=/opt/hadoop-3.1.4
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
source /etc/profile

scp /etc/profile root@node02:/etc/
scp /etc/profile root@node03:/etc/

# 三台机器验证
source /etc/profile
hadoop 
```

### 2.7 格式化启动

#### 2.7.1 初始化集群

```bash
hdfs namenode -format xzh-hadoop
```

#### 2.7.2 单节点启动

```bash
# HDFS集群
hdfs --daemon start namenode|datanode|secondarynamenode
hdfs --daemon stop  namenode|datanode|secondarynamenode
# YARN集群
yarn --daemon start resourcemanager|nodemanager
yarn --daemon stop  resourcemanager|nodemanager
```

```bash
# 192.168.123.201
hdfs --daemon start namenode
hdfs --daemon start datanode
yarn --daemon start resourcemanager
yarn --daemon start nodemanager
# 192.168.123.202
hdfs --daemon start datanode
hdfs --daemon start secondarynamenode
yarn --daemon start nodemanager
# 192.168.123.203
hdfs --daemon start datanode
yarn --daemon start nodemanager
```

#### 2.7.3 一键启动

```bash
cd /opt/hadoop-3.1.4/sbin
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

### 2.8 客户端测试

#### 2.8.1 Web UI

HDFS集群：http://namenode_host:9870

YARN集群：http://resourcemanager_host:8088

#### 2.8.2 功能体验

1. HDFS

```bash
hadoop fs -mkdir /test
hadoop fs -put zookeeper.out /test
hadoop fs -ls /
```

2. MapReduce+YARN 

统计高频词出现次数

```bash
hadoop fs -mkdir -p /wordcount/input
hadoop fs -put a.txt /wordcount/input # 创建a.txt并编辑内容
hadoop jar /opt/hadoop-3.1.4/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.4.jar wordcount /wordcount/input /wordcount/output
hadoop fs -cat /wordcount/output/part-r-00000
```

#### 2.8.3 基准测试

```bash
hadoop jar /opt/hadoop-3.1.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.4-tests.jar TestDFSIO -write -nrFiles 10  -fileSize 10MB
hadoop jar /opt/hadoop-3.1.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.4-tests.jar TestDFSIO -read -nrFiles 10  -fileSize 10MB
hadoop jar /opt/hadoop-3.1.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.4-tests.jar TestDFSIO -clean
```

## 3. shell命令

https://hadoop.apache.org/docs/r3.1.4/hadoop-project-dist/hadoop-common/FileSystemShell.html

```bash
hadoop fs -mkdir /source        # 用于存储原始采集数据
hadoop fs -mkdir /common        # 用于存储公共数据集，例如：IP库、省份信息、经纬度等 
hadoop fs -mkdir /workspace     # 工作空间，存储各团队计算出来的结果数据
hadoop fs -mkdir /tmp           # 存储临时数据，每周清理一次
hadoop fs -mkdir /warehous      # 存储hive数据仓库中的数据

hadoop fs -ls -h -R /tmp        # 查看指定目录下内容 -h 显示size -R 递归
hadoop fs -put a.txt /tmp       # 上传文件
hadoop fs -moveFromLocal  a.txt /tmp    # 上传后删除本地文件
hadoop fs -cat /tmp/a.txt               # 文件查看，大文件慎用
hadoop fs -head /tmp/a.txt              # 查看文件前1KB的内容
hadoop fs -tail -f /tmp/a.txt           # 查看文件后1KB的内容 -f 动态查看
hadoop fs -get /tmp/a.txt  ./           # 下载文件到当前路径 -f 覆盖 -p 保留权限和访问修改时间
hadoop fs -getmerge [-nl] [-skip-empty-file] /tmp/* ./   # 合并下载并再末尾添加换行符
hadoop fs -cp /tmp/a.txt /tmp/b.txt     # 文件拷贝 -f 覆盖
hadoop fs -appendToFile c.txt /tmp/a.txt    # 文件追加
hadoop fs -df -h /tmp                   # 查看hdfs磁盘空间
hadoop fs -du -s -h -v /tmp/a.txt       # 查看文件使用的空间
hadoop fs -mv /tmp/a.txt /source/b.txt  # 文件移动
hadoop fs -setrep -w 2 /tmp/a.txt       # 修改文件副本数 -R 递归 -w 客户端等待副本修改完毕
```

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