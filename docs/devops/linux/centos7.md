# CentOS 7

## 1. 虚拟机

### 1.1 初始化

```bash
yum install -y zip unzip telnet lsof ntpdate openssh-server wget net-tools.x86_64
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
/usr/sbin/ntpdate ntp4.aliyun.com;/sbin/hwclock -w     # 同步时间

service iptables status
systemctl disable iptables.service  # 关闭
systemctl stop firewalld.service
systemctl disable firewalld.service # 关闭
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

### 1.2 OpenSSH

```bash
vi /etc/ssh/sshd_config

# 配置文件
Port 22
ListenAddress 0.0.0.0
ListenAddress ::
PermitRootLogin yes # 允许远程登录
PasswordAuthentication yes  # 开启用户名和密码来验证

# 重启
service sshd start
systemctl enable sshd
```

### 1.3 网络

```bash
vi /etc/hosts                                  # hosts
vi /etc/resolv.conf  nameserver 192.168.0.1    # dns
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3 # ip
hostnamectl set-hostname xuzhihao              # 修改主机名
```

```conf
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static" # dhcp 
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s3"
UUID="e66600c1-35a8-4a09-9bbe-aeafe7ded9b0"
DEVICE="enp0s3"
ONBOOT="yes"
IPV6_PRIVACY="no"
IPADDR=192.168.3.200
NETMASK=255.255.255.0
GATEWAY=192.168.3.1
DNS1=114.114.114.114
```

```
systemctl restart network
```

### 1.4 yum更换

```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo_bak  # 备份本地yum源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo  # 获取阿里yum源配置文件
yum makecache # 更新yum缓存
yum repolist  # 查看当前yum源
```

### 1.5 卸载

```bash
rpm -qa | grep mariadb
rpm -e --nodeps mariadb-libs-5.5.64-1.el7.x86_64
```

### 1.6 vim编辑器

```bash
yum -y install vim*
vi /etc/vimrc  # 添加 colorscheme murphy
vi /etc/profile    # 添加 alias vi=vim
source /etc/profile 
```

```bash
:set nu                 # 显示行号:set nonu
vim +3 /etc/passwd      # 定位到第三行
vim +/sssd /etc/passwd  # 定位到sssd所在的行
```

## 2. 命令

### 2.1 系统

```bash
uname -a                    # 内核信息
cat /proc/version           # 版本信息
cat /etc/redhat-release     # 发行版信息
cat /etc/os-release         # 发行版信息
getconf LONG_BIT            # 查看操作系统位数
env                         # 查看当前用户环境变量

cat /proc/cpuinfo                                       # 查看cpu信息
cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c   # 查看有几个逻辑cpu, 包括cpu型号
cat /proc/cpuinfo | grep physical | uniq -c             # 查看有几颗cpu,每颗分别是几核
cat /proc/cpuinfo | grep flags | grep ' lm ' | wc -l    # 结果大于0, 说明支持64bit计算. lm指long mode, 支持lm则是64bit

echo 3 >/proc/sys/vm/drop_caches    # 清理缓存
HISTTIMEFORMAT='%F %T'              # 历史命令格式化
set +o history                      # 关闭history 
vi /etc/motd                        # 设置欢迎语
timedatectl                             # 查看时区
timedatectl set-timezone Asia/Shanghai  # 设置时区

shutdown -h now # 关机
shutdown -r now # 重启
```

### 2.2 文件

```bash

ls -a
# 替换
sed 's/6379/6380/g' redis-6379.conf > redis-6380.conf
echo 6379 6380 6381 16379 16380 16381 | xargs -t -n 1 cp /usr/local/redis/conf/redis.conf   # 文件批量拷贝至目录

# 压缩
zip -r xzh2021.zip * -x  './node_modules/*'         # 排除指定文件夹
tar -zcvf xzh2021.tar.gz  --exclude=node_modules *  # 排除指定文件夹
tar –zcvf jpg.tar *.jpg             # 压缩
tar –xvf file.tar                   # 解压
tar zxvf file.tar -C /home/data/    # 解压到指定路径

# 拷贝复制
ln -s /usr/local/jdk1.8/ jdk                        # 软连接
cp -f xxx.log   # 复制并强制覆盖同名文件
scp -r vjsp.workflow -P {port} root@20.255.122.15:/opt/code # 远程复制
mkdir -p /home/docker/data  # 级联创建目录
mkdir -p src/{test,main}/{java,resources}   # 批量创建文件夹, 会在test,main下都创建java, resources文件夹

# Find查找
find / -type f -size +100M          # 查找大文件 b/d/c/p/l/f 查是块设备、目录、字符设备、管道、符号链接、普通文件
find / -name memcached              # 查找应用
find / -name 'meeting' -type d      # 查找meeting文件夹所在的位置
find / -name 'server.xml' -print    # 查找server.xml文件的位置
find . -iname 'Configure'           # 查找关键字忽略大小写

find /home/eagleye -name '*.mysql' -print   # 在目录下找后缀是.mysql的文件
find /usr -atime 3 –print                   # 会从/usr目录开始往下找，找最近3天之内存取过的文件。
find /usr -ctime 5 –print                   # 会从/usr目录开始往下找，找最近5天之内修改过的文件。
find /doc -user xzh -name 'j*' –print               # 会从/doc目录开始往下找，找用户xzh的、文件名开头是j的文件。  
find /doc \( -name 'ja*' -o- -name 'ma*' \) –print  # 会从/doc目录开始往下找，找寻文件名是ja开头或者ma开头的文件。
find /doc -name '*bak' -exec rm {} \;               # 会从/doc目录开始往下找，找到凡是文件名结尾为 bak的文件，把它删除掉
```

### 2.3 防火墙

1. iptables

```bash
service iptables status # 查看iptables状态
/etc/init.d/iptables status
/etc/init.d/iptables start
/etc/init.d/iptables stop
/etc/init.d/iptables restart

# -I：添加，-D：删除。INPUT表示入站，***.***.***.*** 表示要封停的IP，DROP表示放弃连接。
iptables -I INPUT -s 211.0.0.0/8 -j DROP    # 封整段
iptables -I INPUT -s 211.1.0.0/16 -j DROP   # 封IP段
iptables -I INPUT -s 61.37.80.0/24 -j DROP
iptables -I INPUT -s 61.37.81.0/24 -j DROP

iptables -D INPUT -s 39.105.58.136 -j DROP        # 解封一个IP
iptables -I INPUT -p tcp --dport 9090 -j ACCEPT   # 开启9090端口的访问
iptables -I INPUT -s 39.105.58.136 -p TCP –dport 80 -j ACCEPT   # 只允许39.105.58.136访问80端口
```

2. firewalld

```bash
systemctl start firewalld.service     # 启动firewall
systemctl restart firewalld.service   # 重启firewall
systemctl stop firewalld.service      # 关闭firewall
systemctl status firewalld.service    # 查看防火墙状态
systemctl disable firewalld.service   # 禁止firewall随系统启动
systemctl enable firewalld.service    # 随系统启动

firewall-cmd --zone=public --add-port=80/tcp --permanent            # 开放端口
firewall-cmd --zone=public --remove-port=9003/tcp --permanent       # 移除端口
firewall-cmd --zone=public --add-port=30000-40000/tcp --permanent   # 批量开放端口
firewall-cmd --query-port=6379/tcp                       # 查看端口是否开启
firewall-cmd --reload                                    # 重启防火墙
```


### 2.4 磁盘

1. 挂载

```bash
du -H -h    # 查看目录及子目录大小
du -sh *    # 查看当前目录下各个文件, 文件夹占了多少空间, 不会递归
sudo fdisk -l              # 查看磁盘空间
fdisk /dev/sdb             # 磁盘分区
m/n/p/1/w
sudo mkfs.ext4 /dev/sdb1   # 格式化磁盘分区
sudo mkdir /data           # 创建挂载点
sudo mount -t ext4 /dev/sdb1 /data   # 磁盘挂载到 /data
df -h

sudo vim /etc/fstab        # 自动挂载
/dev/sdb1 /data ext4 errors=remount-ro 0 1
```

2. 监控

```bash
yum install sysstat iotop -y
iostat -d -x -k 1 10          # -d:设备状态 静态显示每秒的统计（单位kilobytes ） 1s刷新一次，刷新10次
iostat -xz 1
iostat -mx 5 -t >> vmdisk.log 
```

参数说明
```bash
rrqm/s：每秒这个设备相关的读取请求有多少被Merge了（当系统调用需要读取数据的时候，VFS将请求发到各个FS，如果FS发现不同的读取请求读取的是相同Block的数据，FS会将这个请求合并Merge）；
wrqm/s：每秒这个设备相关的写入请求有多少被Merge了。
r/s： 该设备的每秒完成的读请求数（merge合并之后的）
w/s:  该设备的每秒完成的写请求数（merge合并之后的）
rsec/s：每秒读取的扇区数；
wsec/：每秒写入的扇区数。
rKB/s：每秒发送给该设备的总读请求数 
wKB/s：每秒发送给该设备的总写请求数 
avgrq-sz 平均请求扇区的大小
avgqu-sz 是平均请求队列的长度。毫无疑问，队列长度越短越好。    
await：  每一个IO请求的处理的平均时间（单位是微秒毫秒）。这里可以理解为IO的响应时间，一般地系统IO响应时间应该低于5ms，如果大于10ms就比较大了。这个时间包括了队列时间和服务时间，也就是说，一般情况下，await大于svctm，它们的差值越小，则说明队列时间越短，反之差值越大，队列时间越长，说明系统出了问题。
svctm:    表示平均每次设备I/O操作的服务时间（以毫秒为单位）。如果svctm的值与await很接近，表示几乎没有I/O等待，磁盘性能很好，如果await的值远高于svctm的值，则表示I/O队列等待太长，系统上运行的应用程序将变慢。
%util： 在统计时间内所有处理IO时间，除以总共统计时间。例如，如果统计间隔1秒，该设备有0.8秒在处理IO，而0.2秒闲置，那么该设备的%util = 0.8/1 = 80%，所以该参数暗示了设备的繁忙程度。一般地，如果该参数是100%表示设备已经接近满负荷运行了（当然如果是多磁盘，即使%util是100%，因为磁盘的并发能力，所以磁盘使用未必就到了瓶颈）。
```

### 2.5 网络

1. 进程

```bash
ps -aux | grep redis          # 查看启动进程参数
lsof -i:80                    # 可以看到pid和用户 
netstat -tunlp | grep 8080    # 端口占用查看
netstat -anp | grep 17010pos 
/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"  # 获取本机ip地址

# 输出每个ip的连接数，以及总的各个状态的连接数
netstat -n | awk '/^tcp/ {n=split($(NF-1),array,":");if(n<=2)++S[array[(1)]];else++S[array[(4)]];++s[$NF];++N} END {for(a in S){printf("%-20s %s\n", a, S[a]);++I}printf("%-20s %s\n","TOTAL_IP",I);for(a in s) printf("%-20s %s\n",a, s[a]);printf("%-20s %s\n","TOTAL_LINK",N);}'
# 统计所有连接状态
# CLOSED：无连接是活动的或正在进行
# LISTEN：服务器在等待进入呼叫
# SYN_RECV：一个连接请求已经到达，等待确认
# SYN_SENT：应用已经开始，打开一个连接
# ESTABLISHED：正常数据传输状态
# FIN_WAIT1：应用说它已经完成
# FIN_WAIT2：另一边已同意释放
# ITMED_WAIT：等待所有分组死掉
# CLOSING：两边同时尝试关闭
# TIME_WAIT：主动关闭连接一端还没有等到另一端反馈期间的状态
# LAST_ACK：等待所有分组死掉
netstat -n | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}'

# 查找较多time_wait连接
netstat -n|grep TIME_WAIT|awk '{print $5}'|sort|uniq -c|sort -rn|head -n20
traceroute -I www.163.com           # traceroute默认使用udp方式, 如果是-I则改成icmp方式
traceroute -M 3 www.163.com         # 从ttl第3跳跟踪
traceroute -p 8080 192.168.10.11    # 加上端口跟踪
```

2. TCP调试

```bash
nc -z -w 3 192.168.20.183 7443 && echo ok || echo not ok
nc -v -w 10 -z 192.168.20.183 7443
nc -v -w 2 -z 127.0.0.1 7000-7500

```

3. 流量监控

```bash
wget http://gael.roualland.free.fr/ifstat/ifstat-1.1.tar.gz # 下载
tar -zxvf ifstat-1.1.tar.gz
cd ifstat-1.1
./configure            # 默认会安装到/usr/local/bin/目录中
make;make install
ifstat -tT
```

参数
```
-l 监测环路网络接口（lo）。缺省情况下，ifstat监测活动的所有非环路网络接口。经使用发现，加上-l参数能监测所有的网络接口的信息，而不是只监测 lo的接口信息，也就是说，加上-l参数比不加-l参数会多一个lo接口的状态信息。
-a 监测能检测到的所有网络接口的状态信息。使用发现，比加上-l参数还多一个plip0的接口信息，搜索一下发现这是并口（网络设备中有一 个叫PLIP (Parallel Line Internet Protocol). 它提供了并口...）
-z 隐藏流量是无的接口，例如那些接口虽然启动了但是未用的
-i 指定要监测的接口,后面跟网络接口名
-s 等于加-d snmp:[comm@][#]host[/nn]] 参数，通过SNMP查询一个远程主机
-h 显示简短的帮助信息
-n 关闭显示周期性出现的头部信息（也就是说，不加-n参数运行ifstat时最顶部会出现网络接口的名称，当一屏显示不下时，会再一次出现接口的名称，提示我们显示的流量信息具体是哪个网络接口的。加上-n参数把周期性的显示接口名称关闭，只显示一次）
-t 在每一行的开头加一个时间 戳（能告诉我们具体的时间）
-T 报告所有监测接口的全部带宽（最后一列有个total，显示所有的接口的in流量和所有接口的out流量，简单的把所有接口的in流量相加,out流量相 加）
-w  用指定的列宽，而不是为了适应接口名称的长度而去自动放大列宽
-W 如果内容比终端窗口的宽度还要宽就自动换行
-S 在同一行保持状态更新（不滚动不换行）注：如果不喜欢屏幕滚动则此项非常方便，与bmon的显示方式类似
-b 用kbits/s显示带宽而不是kbytes/s
-q 安静模式，警告信息不出现
-v 显示版本信息
-d 指定一个驱动来收集状态信息
```

### 2.6 环境 

1. gcc

```bash
yum -y install centos-release-scl
sudo yum install devtoolset-7-gcc*
sudo yum install devtoolset-9-gcc* 
scl enable devtoolset-7 bash
scl enable devtoolset-9 bash
echo "source /opt/rh/devtoolset-9/enable" >> /etc/profile
which gcc
gcc --version
```

### 2.7 应用

1. 启动命令

```bash
# solr
cd /opt/solr-4.7.2/example
nohup java -Djetty.port=8080 -jar start.jar &
ps aux|grep jetty

# openfire
cd /opt/openfire/bin
nohup ./openfire.sh >/dev/null 2>&1 &

# sentinel
nohup java -Dserver.port=9000 -jar sentinel-dashboard-1.7.2.jar >out.log 2>&1 &
```

2. FTP安装

```bash
yum install vsftpd                  # 安装
systemctl restart vsftpd.service    # 重启
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config     # 关闭selinux
cat /etc/vsftpd/vsftpd.conf         # 配置文件
cat /etc/vsftpd/vsftpd.conf | grep -v "#" | grep -v "^$" > /etc/vsftpd/vsftpd.conf.bak

vim /etc/vsftpd/ftpusers    # 连接黑名单，总是生效
vim /etc/vsftpd/user_list   # 自定义黑名单，对应配置文件中 userlist_enable=YES 选项和 userlist_file 的值，默认：userlist_file=/etc/vsftpd/user_list

cat /etc/passwd       # 查看用户
useradd xzh -g xzh -d /opt/xzh.webapp -s /sbin/nologin
passwd xzh
chmod -R 777 /opt/xzh.webapp
userdel xzh

# -s /sbin/nologin 无法登录需要修改
vim /etc/shells
/sbin/nologin
```

## 3. 快捷键

```bash
ctrl + z / fg                       # 挂起
ctrl + s / q                        # 锁屏
ctrl + u                            # 前删除
ctrl + k                            # 后删除
Ctrl + Shift + c                    # 复制
Ctrl + Shift + v                    # 粘贴    
```

## 4. 开发环境

### 4.1 Java

```bash
# jdk
yum install java-1.8.0-openjdk* -y
yum install java-1.8.0-openjdk

vim /etc/profile
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export PATH=$PATH:$JAVA_HOME/bin
source /etc/profile   # 配置生效

# maven
tar -xzf apache-maven-3.6.2-bin.tar.gz    # 解压
mkdir -p /opt/maven                       # 创建目录
mv apache-maven-3.6.2/* /opt/maven        # 移动文件

vi /etc/profile
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export MAVEN_HOME=/opt/maven
export PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin
source /etc/profile   # 配置生效
mvn -v                # 查找Maven版本
```

```xml
  <localRepository>/opt/repository</localRepository>
  <mirrors>
        <!-- 阿里云仓库 -->
        <mirror>
            <id>alimaven</id>
            <mirrorOf>central</mirrorOf>
            <name>aliyun maven</name>
            <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
        </mirror>
        <!-- 中央仓库1 -->
        <mirror>
            <id>repo1</id>
            <mirrorOf>central</mirrorOf>
            <name>Human Readable Name for this Mirror.</name>
            <url>http://repo1.maven.org/maven2/</url>
        </mirror>
        <!-- 中央仓库2 -->
        <mirror>
            <id>repo2</id>
            <mirrorOf>central</mirrorOf>
            <name>Human Readable Name for this Mirror.</name>
            <url>http://repo2.maven.org/maven2/</url>
        </mirror>
  </mirrors>
```


### 4.2 Node

```bash
yum install -y git
wget https://nodejs.org/dist/v14.16.0/node-v14.16.0-linux-x64.tar.xz
tar -xvf node-v14.16.0-linux-x64.tar.xz
mv node-v14.16.0-linux-x64 /usr/local/node

vim /etc/profile
export NODE_HOME=/usr/local/node
export PATH=$NODE_HOME/bin:$PATH

source /etc/profile
node -v
npm -v
```

```bash
npm install forever -g      #全局安装forever启动命令
forever start app.js        #启动进程
forever stop  app.js        #关闭进程
forever stopall             #关闭所有进程
forever restart app.js      #重启进程
forever list                #查看服务进程
forever start -w app.js     #监听文件改动
forever start -l forever.log -o out.log -e err.log app.js #日志输出
```


### 4.3 Npm

```bash
npm -v #查看npm安装的版本

npm install --registry=https://registry.npm.taobao.org #指定仓库地址

npm init                        #创建package.json
npm install moduleName          #安装node模块
npm install moduleName@1.0.0    #安装node模块特定版本
npm install -g moduleName       #全局安装命令
npm install –save               #将模块写入dependencies节点（生产环境）
npm install –save-dev/          #将模块写入devDependencies节点（开发环境）
npm set global=true             #设定全局安装模式
npm get global                  #查看当前使用的安装模式
npm outdated                    #检查包是否已经过时
npm update moduleName           #更新node模块
npm uninstall moudleName        #卸载node模块

npm root                #查看当前包的安装路径
npm root -g             #查看全局的包的安装路径
npm list                #查看当前目录下已安装的node包
npm list parseable=true #以目录的形式来展现当前安装的所有node包
```

### 4.4 TypeScript

```bash
npm init -y                     # 生成package.json配置文件
npm install -g typescript --registry=https://registry.npm.taobao.org # 全局安装ts，如果安装过了，忽略这个命令
tsc --init                      # 生成tsconfig.json配置文件
npm install rollup typescript rollup-plugin-typescript2 "@rollup/plugin-node-resolve" rollup-plugin-serve -D  # 安装rollup环境依赖
tsc -w                          # 手动编译
npm install ts-node -g --force  # 配合插件Code Runner
```

### 4.5 Golang

```bash
wget  https://dl.google.com/go/go1.13.4.linux-amd64.tar.gz          # 下载
tar zxvf go1.13.4.linux-amd64.tar.gz
mv go /usr/local/
echo 'export PATH=$PATH:/usr/local/go/bin'>>/etc/profile            # SDK
echo 'export GOPATH=/home/xzh/go'>>/etc/profile                     # 工作空间
echo 'export GOBIN=/home/xzh/go/bin'>>/etc/profile                  # 生成可执行文件路径
echo 'export GO111MODULE=on'>>/etc/profile                          # 模块支持
echo 'export GOPROXY=https://goproxy.cn'>>/etc/profile              # 模块代理
echo 'export http_proxy=http://172.17.17.165:7890'>>/etc/profile    # 翻墙
echo 'export https_proxy=http://172.17.17.165:7890'>>/etc/profile
source /etc/profile

go version
go env
```

## 5. Shell

### 5.1 Tomcat监控 

1. 重启

restart_3001.sh
```bash
sh /data/tomcat_webapp_3001/bin/shutdown.sh
sleep 2s
ps -ef | grep tomcat_webapp_3001 | grep -v grep | awk '{print $2}'| xargs kill -9
sleep 1s
sh /data/tomcat_webapp_3001/bin/startup.sh;tail -f /data/tomcat_webapp_3001/logs/catalina.out
```

2. war部署

deploy.sh
```bash
sh /opt/tomcat/bin/shutdown.sh
sleep 2s
ps -ef | grep /opt/tomcat/ | grep -v grep | awk '{print $2}'| xargs kill -9
sleep 1s
rm -rf /opt/tomcat/webapps/servlet*
cp -r /opt/tomcat/code/servlet-2.war /opt/tomcat/webapps/servlet.war
sh /opt/tomcat/bin/startup.sh;tail -f /opt/tomcat/logs/catalina.out
```

### 5.2 Spring Boot启动

启动run.sh

```bash
#!/bin/bash

JAVA_OPTS="-server -Xms1024m -Xmx1024m -Xmn1024m -XX:MetaspaceSize=1024m -XX:MaxMetaspaceSize=1024m -Xverify:none -XX:+DisableExplicitGC -Djava.awt.headless=true"

jar_name="eureka-server.jar"
this_dir="$( cd "$( dirname "$0"  )" && pwd )"
log_dir="${this_dir}/logs"
jar_file="${this_dir}/${jar_name}"
echo "${jar_file}"

#日志文件夹不存在，则创建
if [ ! -d "${log_dir}" ]; then
    mkdir "${log_dir}"
fi

#父目录下jar文件存在
if [ -f "${jar_file}" ]; then
   	nohup java $JAVA_OPTS -jar ${jar_file} 1> ${log_dir}/catalina.log 2>&1 &
    exit 0
else
    echo -e "\033[31m${jar_file}文件不存在！\033[0m"
    exit 1
fi
```

```bash
sed -i 's/\r$//' run.sh  
```

### 5.3 Jdk批量

```bash
vim /etc/hosts
192.168.3.200 node01 node01.hadoop.com
192.168.3.201 node02 node02.hadoop.com
192.168.3.202 node03 node03.hadoop.com
```

```bash
# A机器执行
cd /root/.ssh
ssh-keygen -t rsa
scp /root/.ssh/id_rsa.pub root@192.168.3.201:/root/.ssh/authorized_keys
scp /root/.ssh/id_rsa.pub root@192.168.3.202:/root/.ssh/authorized_keys

# BC机器执行
cd /root/.ssh
cat id_rsa.pub >>authorized_keys
chmod 600 authorized_keys
```

vi install_jdk.sh

```bash
#!/bin/bash
tar -zxvf /export/softwares/jdk-8u141-linux-x64.tar.gz -C /export/servers/
cd /export/servers/jdk1.8.0_141
home=`pwd`
echo $home
echo "export JAVA_HOME=${home}"  >> /etc/profile
echo "export PATH=:\$PATH:\$JAVA_HOME/bin" >> /etc/profile
source /etc/profile
for m in  2 3
do
scp -r /export/servers/jdk1.8.0_141 node0$m:/export/servers/
ssh node0$m "echo 'export JAVA_HOME=/export/servers/jdk1.8.0_141' >> /etc/profile; echo 'export PATH=:\$PATH:\$JAVA_HOME/bin' >> /etc/profile;source /etc/profile"
done
```