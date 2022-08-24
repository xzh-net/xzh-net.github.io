# Hadoop 3.1.4

## 1.编译

### 1.1 安装依赖

```bash
yum install gcc gcc-c++
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

#### 2.2.1 修改主机名

```bash
hostnamectl set-hostname node01.xuzhihao.net
hostnamectl set-hostname node02.xuzhihao.net
hostnamectl set-hostname node03.xuzhihao.net
```

#### 2.2.2 hosts映射

```bash
vim /etc/hosts
# 添加
192.168.2.201 node01 node01.xuzhihao.net
192.168.2.202 node02 node02.xuzhihao.net
192.168.2.203 node03 node03.xuzhihao.net
```

#### 2.2.3 关闭防火墙

```bash
systemctl stop firewalld.service 
systemctl disable firewalld.service
```

#### 2.2.4 免密登录

```bash
ssh-keygen # 3个回车 生成公钥、私钥
# 192.168.2.201 执行
ssh-copy-id node02
ssh-copy-id node03
ssh-copy-id node01
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
```

### 2.4 修改配置

#### 2.4.1 hadoop-env.sh

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim hadoop-env.sh
# 编辑内容
export JAVA_HOME=/usr/local/jdk1.8.0_202
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
# 此处为节点下线做准备
<property>
	<name>dfs.hosts.exclude</name>
	<value>/opt/hadoop-3.1.4/etc/hadoop/excludes</value>
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

#### 2.4.6 workers

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
scp -r /opt/hadoop-3.1.4 root@node02:$PWD
scp -r /opt/hadoop-3.1.4 root@node03:$PWD
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
hdfs namenode -format
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
# 192.168.2.201
hdfs --daemon start namenode
hdfs --daemon start datanode
yarn --daemon start resourcemanager
yarn --daemon start nodemanager
# 192.168.2.202
hdfs --daemon start datanode
hdfs --daemon start secondarynamenode
yarn --daemon start nodemanager
# 192.168.2.203
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

### 2.9 动态扩容

#### 2.9.1 基础环境准备

见2.2 基础环境准备，在新机器完成以上操作

#### 2.9.2 内容分发

1. 环境变量

```bash
# 192.168.2.201执行
scp /etc/profile root@node04:/etc/
```

2. 软件包

```bash
# 192.168.2.201执行
cd /opt/hadoop-3.1.4
scp -r /opt/hadoop-3.1.4 root@node04.xuzhihao.net:$PWD
```

#### 2.9.3 启动节点

```bash
hdfs --daemon start namenode
```

#### 2.9.4 DataNode负载均衡

```bash
# 在主节点执行
hdfs dfsadmin -setBalancerBandwidth 104857600   # 设置带宽100M
hdfs balancer -threshold 5                      # 均衡比例
```

#### 2.9.5 Web页面查看情况

### 2.10 节点退役

#### 2.10.1 修改dfs.hosts.exclude

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim exclude

# 添加下线节点
node04.xuzhihao.net
```

!>注意：如果副本数是3，服役的节点小于等于3，是不能退役成功的，需要修改副本数后才能退役

#### 2.10.2 节点刷新

```bash
# 在namenode节点执行
hdfs dfsadmin -refreshNodes
```

Web页面查看情况等待节点状态变成decommissioned表示所有块已经复制完毕

#### 2.10.3 关闭节点

```bash
hdfs --daemon stop datanode
hdfs balancer -threshold 5      # 重新执行负载均衡
```

## 3. HDFS HA

![](../../assets/_images/devops/deploy/hadoop/1.png)

### 3.1 集群规划

```lua
node01  namenode  zkfc  datanode  zookeeper  journal node
node02  namenode  zkfc  datanode  zookeeper  journal node
node03                  datanode  zookeeper  journal node
```

基础环境准备见2.2 

### 3.2 安装zookeeper集群

!>需要设置免密登录

### 3.3 上传安装包

#### 3.3.1 解压

```bash
cd /opt/software
tar -zxvf hadoop-3.1.4-bin-snappy-CentOS7.tar.gz -C /opt
```

#### 3.3.2 创建目录

```bash
mkdir -p /opt/software/     # 安装包存放路径，已创建
mkdir -p /opt/              # 软件安装路径，已创建
mkdir -p /opt/journaldata/  # 数据存储路径
```

### 3.4 修改配置

#### 3.4.1 hadoop-env.sh

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim hadoop-env.sh
# 编辑内容
export JAVA_HOME=/usr/local/jdk1.8.0_202
# 添加到末尾，设置用户以执行对应角色shell命令
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_JOURNALNODE_USER=root
export HDFS_ZKFC_USER=root
```

#### 3.4.2 core-site.xml

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim core-site.xml

# 添加到configuration区间
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://mycluster</value>
</property>

<!-- hadoop数据存储目录-->
<property>
    <name>hadoop.tmp.dir</name>
    <value>/opt/hadoop-3.1.4/tmp</value>
</property>

<!-- ZooKeeper集群的地址和端口-->
<property>
    <name>ha.zookeeper.quorum</name>
    <value>node01.xuzhihao.net:2181,node02.xuzhihao.net:2181,node03.xuzhihao.net:2181</value>
</property>

<!-- 在Web UI访问HDFS使用的用户名。-->
<property>
    <name>hadoop.http.staticuser.user</name>
    <value>root</value>
</property>
```

#### 3.4.3 hdfs-site.xml

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim hdfs-site.xml

# 添加到configuration区间
<!--指定hdfs的nameservice为mycluster，需要和core-site.xml中的保持一致 -->
<property>
    <name>dfs.nameservices</name>
    <value>mycluster</value>
</property>
<!-- mycluster下面有两个NameNode，分别是nn1，nn2 -->
<property>
    <name>dfs.ha.namenodes.mycluster</name>
    <value>nn1,nn2</value>
</property>

<!-- nn1的RPC通信地址 -->
<property>
    <name>dfs.namenode.rpc-address.mycluster.nn1</name>
    <value>node01.xuzhihao.net:8020</value>
</property>

<!-- nn1的http通信地址 -->
<property>
    <name>dfs.namenode.http-address.mycluster.nn1</name>
    <value>node01.xuzhihao.net:9870</value>
</property>

<!-- nn2的RPC通信地址 -->
<property>
    <name>dfs.namenode.rpc-address.mycluster.nn2</name>
    <value>node02.xuzhihao.net:8020</value>
</property>

<!-- nn2的http通信地址 -->
<property>
    <name>dfs.namenode.http-address.mycluster.nn2</name>
    <value>node02.xuzhihao.net:9870</value>
</property>

<!-- 指定NameNode的edits元数据在JournalNode上的存放位置 -->
<property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://node01.xuzhihao.net:8485;node02.xuzhihao.net:8485;node03.xuzhihao.net:8485/mycluster</value>
</property>

<!-- 指定JournalNode在本地磁盘存放数据的位置 -->
<property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/opt/journaldata</value>
</property>

<!-- 开启NameNode失败自动切换 -->
<property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
</property>

<!-- 指定该集群出故障时，哪个实现类负责执行故障切换 -->
<property>
    <name>dfs.client.failover.proxy.provider.mycluster</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>

<!-- 配置隔离机制方法-->
<property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
</property>

<!-- 使用sshfence隔离机制时需要ssh免登陆 -->
<property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/root/.ssh/id_rsa</value>
</property>

<!-- 配置sshfence隔离机制超时时间 -->
<property>
    <name>dfs.ha.fencing.ssh.connect-timeout</name>
    <value>30000</value>
</property>
```

!> 远程补刀需要namenode互相免密登录，同时安装psmisc

```bash
ssh-copy-id node01  # 在node02节点执行
yum install psmisc -y
```

#### 3.4.4 workers

```bash
cd /opt/hadoop-3.1.4/etc/hadoop/
vim workers

# 删除第一行localhost，然后添加以下三行
node01.xuzhihao.net
node02.xuzhihao.net
node03.xuzhihao.net
```

### 3.5 分发安装包

```bash
cd /opt/hadoop-3.1.4
scp -r /opt/hadoop-3.1.4 root@node02:$PWD
scp -r /opt/hadoop-3.1.4 root@node03:$PWD
```

### 3.6 启动集群

#### 3.6.1 启动zk

```bash
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh start
/opt/apache-zookeeper-3.7.0/bin/zkServer.sh status
```

#### 3.6.2 启动journalnode

```bash
hdfs --daemon start journalnode # 三台机器
```
			
#### 3.6.3 格式化namenode

```bash
hdfs namenode -format xzh-hadoop    # node01初始化集群
hdfs --daemon start namenode        # node01启动namenode
hdfs namenode -bootstrapStandby     # node02同步元数据
```

#### 3.6.4 格式化ZKFC

```bash
hdfs zkfc -formatZK     # active上执行
```

#### 3.6.5 启动HDFS

```bash
cd /opt/hadoop-3.1.4/sbin
start-dfs.sh
```

#### 3.6.6 启动YARN（可选）

#### 3.6.7 检查集群状态

```bash
hdfs dfsadmin -report
```

## 4. shell命令

https://hadoop.apache.org/docs/r3.1.4/hadoop-project-dist/hadoop-common/FileSystemShell.html

### 4.1 hdfs

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

### 4.2 集群

```bash
hdfs dfsadmin -safemode get     # 手动获取安全模式状态信息
hdfs dfsadmin -safemode enter   # 进入安全模式
hdfs dfsadmin -safemode leave   # 离开安全模式


```

## 5. 组件

### 5.1 WebHDFS

HDFS Web UI，其文件浏览功能底层就是基于WebHDFS来操作HDFS实现的

?> 官方参数说明：https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Open_and_Read_a_File

#### 5.1.1 使用示例

1. 查看目录下文件

```bash
http://192.168.2.201:9870/webhdfs/v1/?op=LISTSTATUS
```

该操作表示要查看根目录下的所有文件以及目录，相当于 `hadoop fs -ls /`

2. 读取指定文件内容

```bash
http://192.168.2.201:9870/webhdfs/v1/test/a.txt?op=OPEN&noredirect=false
```

3. 上传文件

发送put请求

```bash
http://192.168.2.201:9870/webhdfs/v1/test/b.txt?op=CREATE&overwrite=true&replication=2&noredirect=true
```

点击连接跳转，body参数选择二进制，选择上传文件

### 5.2 HttpFS 网关代理服务

#### 5.2.1 开启服务

1. 修改core-site.xml

```bash
vi /opt/hadoop-3.1.4/etc/hadoop/core-site.xml
# 添加
<property>
    <name>hadoop.proxyuser.root.hosts</name>
    <value>*</value>
</property>
<property>
    <name>hadoop.proxyuser.root.groups</name>
    <value>*</value>
</property>
```

2. 分发重启服务

```bash
cd /opt/hadoop-3.1.4/etc/hadoop
scp core-site.xml node02:/opt/hadoop-3.1.4/etc/hadoop/
scp core-site.xml node03:/opt/hadoop-3.1.4/etc/hadoop/

cd /opt/hadoop-3.1.4/sbin
./stop-all.sh
./start-all.sh

hdfs --daemon start httpfs  # 启动httpfs
```

#### 5.2.2 访问地址

http://192.168.2.201:14000/static/index.html

http://192.168.2.201:14000/webhdfs/v1?user.name=root&op=LISTSTATUS