# CentOS Linux release 7.9.2009

- https://ftp.redhat.com
- https://www.rpmfind.net
- https://rpm.pbone.net

## 1. 服务

### 1.1 Yum

#### 1.1.1 备份yum源

```bash
cd /etc/yum.repos.d/ && mkdir bakup && mv *.repo bakup
```

#### 1.1.2 使用光驱挂载本地yum源

```bash
lsblk                       # 查看可用设备信息
mkdir /mnt/cdrom            # 创建挂载文件夹
mount /dev/cdrom /mnt/cdrom # 创建挂载点
echo "/dev/cdrom /mnt/cdrom iso9660 defaults 0 0" >> /etc/fstab # 永久挂载

yum install -y psmisc   # 无法删除挂载点需要使用kuser命令
fuser -km /mnt/cdrom    # kill 挂载进程
umount /mnt/cdrom       # 取消挂载
```

#### 1.1.3 配置本地yum源

```bash
cd /etc/yum.repos.d/
vi local-yum.repo
# 添加以下内容
[local-yum]
name=local yum
baseurl=file:///mnt/cdrom
enabled=1
gpgcheck=0
```

#### 1.1.4 配置网络yum源

```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo_bak              # 备份本地yum源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo  # 获取阿里yum源配置文件
yum repolist    # 查看源信息
yum clean all   # 清空缓存  
yum makecache   # 更新yum缓存
```

### 1.2 DHCP

- DHCP主机的IP为: 192.168.100.1/24
- DHCP动态分配的IP范围为： 192.168.100.100/24 - 192.168.100.200/24
- DHCP客户端的网关设置为:  192.168.100.1

#### 1.2.1 配置IP

```bash
vi /etc/sysconfig/network-scripts/ifcfg-enps33

IPADDR=192.168.100.1
NETMASK=255.255.255.0
GATEWAY=192.168.100.1

systemctl restart network
ifconfig
```

#### 1.2.2 安装

```bash
yum -y install dhcp
rpm -ql dhcp

/etc/dhcp
/etc/dhcp/dhcpd.conf    # 主配置文件
/etc/rc.d/init.d/dhcpd  # 启动脚本
/usr/sbin/dhcpd         # 二进制命令
```

#### 1.2.3 修改配置

?> 服务器的地址必须与仅主机模式中设置的ip网段相同

```bash
vim /etc/dhcp/dhcpd.conf

default-lease-time 259200;    # 预设租期3天
max-lease-time 518400;        # 最大租期6天
option domain-name "xuzhihao.com";         # 指定默认域名
option domain-name-servers 192.168.100.1;  # DNS(可以写多个)
ddns-update-style none;       # 禁用 DNS 动态更新 
log-facility local7;          # 定义日志设备载体 （/var/log/boot.log输出）

subnet 192.168.100.0 netmask 255.255.255.0 {    # 子网
  range 192.168.100.100 192.168.100.200;        # 地址范围
  option routers 192.168.100.1;                 # 网关，客户端的默认的转发地址
  option broadcast-address 192.168.2.255;       # 广播地址
}

```

#### 1.2.4 启动服务

```bash
systemctl start dhcpd
systemctl enable dhcpd
```

#### 1.2.5 客户端测试

client端修改IP地址为动态获取

```bash
vi /etc/sysconfig/network-scripts/ifcfg-enps33

BOOTPROTO=dhcp
#IPADDR=192.168.3.200
#NETMASK=255.255.255.0
#GATEWAY=192.168.3.1
#DNS1=114.114.114.114

systemctl restart network
ifconfig
```

### 1.3 DNS

#### 1.3.1 正向解析

1. 安装

```bash
yum -y install bind
rpm -q bind     # 确认安装成功
rpm -ql bind    # 查看安装文件列表
```

2. 配置文件

```bash
/etc/logrotate.d/named      # 日志轮转文件
/etc/named                  # 配置文件的主目录
/etc/named.conf             # 主配置文件
/etc/named.rfc1912.zones    # zone文件，定义域

/usr/sbin/named             # 二进制命令
/usr/sbin/named-checkconf   # 检查配置文件的命令
/usr/sbin/named-checkzone   # 检查区域文件的命令

/var/log/named.log          # 日志文件
/var/named                  # 数据文件的主目录
/var/named/data             # 
/var/named/dynamic          # 
/var/named/named.ca         # 根域服务器
/var/named/named.empty      # 
/var/named/named.localhost  # 正向解析区域文件的模板
/var/named/named.loopback   # 反向解析区域文件的模板
/var/named/slaves           # 从dns服务器下载文件的默认路径
/var/run/named              # 进程文件
```

3. 修改主配置文件

```bash
cp -p /etc/named.conf /etc/named.conf.bak
vim /etc/named.conf

options {
        listen-on port 53 { 127.0.0.1;any; };   # 添加any
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { localhost;any; };   # 添加any

        recursion yes;
        dnssec-enable no;
        dnssec-validation no;

        bindkeys-file "/etc/named.root.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

```

4. 修改子配置文件

```bash
cp -p /etc/named.rfc1912.zones /etc/named.rfc1912.zones.bak
vim /etc/named.rfc1912.zones

# 在该文件最后面增加以下内容：
zone "hwcq.online" IN {
        type master;
        file "hwcq.online.zone";
        allow-update { none; };
};
```

5. 配置zone文件

```bash
cp -p /var/named/named.localhost /var/named/hwcq.online.zone    # 一定要使用-p复制用户组和权限
vi /var/named/hwcq.online.zone

$TTL 1D
@       IN SOA  hwcq.online.  rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@       NS      dns1.hwcq.online.   # dns1表示命名空间，可以有多个，但是后面的Ａ记录保持一致就行
dns1    A       127.0.0.1           # 一定是当前DNS服务器的IP
www     A    192.168.3.200          # 根据实际填写
```

6. 检查配置文件的语法错误

```bash
named-checkconf /etc/named.conf
named-checkconf /etc/named.rfc1912.zones

cd /var/named/
named-checkzone hwcq.online.zone hwcq.online.zone # 区域文件写2遍
```

7. 启动服务

```bash
systemctl start named
netstat -tunlp|grep 53  # 端口
systemctl enable named
```


8. 客户端测试

```bash
echo nameserver 172.17.17.201 > /etc/resolv.conf # 客户端机器添加dns服务器

nslookup www.hwcq.online
dig @172.17.17.201 www.hwcq.online
host www.hwcq.online
```


#### 1.3.2 反向解析

1. 修改子配置文件（下面操作基于正向解析环境）

```bash
vim /etc/named.rfc1912.zones

# 在该文件最后面增加以下内容：
zone "17.17.172.in-addr.arpa" IN {
	type master;
	file "172.17.17.zone";
	allow-update { none; };
};
```

2. 配置zone文件

```bash
cp -p /var/named/named.loopback /var/named/172.17.17.zone    # 一定要使用-p复制用户组和权限
vi /var/named/172.17.17.zone

$TTL 1D
@	IN SOA	hwcq.online. rname.invalid. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum
@	NS	dns2.hwcq.online.   # 如果dns2.hwcq.online在正向域文件中存在，可以不用写A记录
dns2	A	127.0.0.1
165	PTR	www.hwqc.online.
```

3. 检查配置文件的语法错误

```bash
named-checkconf /etc/named.conf
named-checkconf /etc/named.rfc1912.zones

cd /var/named/
named-checkzone 172.17.17.zone 172.17.17.zone # 区域文件写2遍
```

4. 启动服务

```bash
systemctl restart named
systemctl status named
```

5. 客户端测试

```bash
echo nameserver 172.17.17.201 > /etc/resolv.conf # 客户端机器添加dns服务器

nslookup 172.17.17.165
dig @172.17.17.201 -x 172.17.17.165
host 172.17.17.165
```


#### 1.3.3 主从搭建

主从的系统时间要保持一致，需要提前准备好时间同步服务器，并做好配置，主从并不是高可用，而是在客户端配置了多套dns地址

1. 同步master和slave的系统时间

```bash
crontab -e
*/2 * * * * /usr/sbin/ntpdate ntp4.aliyun.com &>/dev/null
```

2. 搭建dns服务器slave节点

```bash
yum -y install bind
```

修改主配置

```bash
vi /etc/named.conf
# 修改内容
options {
	listen-on port 53 { 127.0.0.1;any; };
	#listen-on-v6 port 53 { ::1; };
	allow-query     { localhost;any; };
	
    dnssec-enable no;
	dnssec-validation no;
};
```

修改子配置

```bash
vi /etc/named.rfc1912.zones
# 最后添加
zone "hwcq.online" IN {
        type slave;
        masters {172.17.17.201;};       # 指定master dns的ip地址
        file "slaves/hwcq.online.zone"; # 同步过来的文件的保存路径及名字
};
```

3. 修改dns服务器master节点

```bash
vi /etc/named.rfc1912.zones

# 修改以下内容：
zone "hwcq.online" IN {
        type master;
        file "hwcq.online.zone";
        //allow-update { none; }; # 删除
};
```

4. 重启主备服务

```bash
systemctl restart named
systemctl status named
```

5. 验证slave节点

```bash
cd /var/named/slaves
ll  # 可见同步过来的hwcq.online.zone文件
```

6. 客户端测试

```bash
echo nameserver 172.17.17.201 > /etc/resolv.conf  # 客户端机器添加dns服务器
echo nameserver 172.17.17.200 >> /etc/resolv.conf 

nslookup www.hwcq.online
dig @172.17.17.201 www.hwcq.online
host www.hwcq.online
```

### 1.4 SSH

#### 1.4.1 配置文件

```bash
vi /etc/ssh/sshd_config

# 配置文件
Port 22
ListenAddress 0.0.0.0
ListenAddress ::
PermitRootLogin yes # 允许远程登录
PasswordAuthentication yes  # 开启用户名和密码来验证

systemctl start  sshd   # 启动服务
systemctl enable sshd   # 开机自启
```

#### 1.4.2 免密登录

```bash
# 192.168.3.201机器执行
ssh-keygen -t rsa
cd /root/.ssh
ssh-copy-id -i id_rsa.pub root@192.168.3.202
ssh-copy-id -i id_rsa.pub root@192.168.3.203

# 192.168.3.202,192.168.3.203 如果提示没有权限
cd /root/.ssh
chmod 600 authorized_keys
```

### 1.5 FTP

#### 1.5.1 安装

```bash
yum install vsftpd
```

#### 1.5.2 修改配置

```bash
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config     # 关闭selinux
cp -p /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak       # 备份
cat /etc/vsftpd/vsftpd.conf.bak | grep -v "#" | grep -v "^$" > /etc/vsftpd/vsftpd.conf  # 去掉注释

vim /etc/vsftpd/ftpusers    # 连接黑名单，总是生效
vim /etc/vsftpd/user_list   # 自定义黑名单，对应配置文件中 userlist_enable=YES 选项和 userlist_file 的值，默认：userlist_file=/etc/vsftpd/user_list

# -s /sbin/nologin 无法登录需要修改
vim /etc/shells
/sbin/nologin
```

#### 1.5.3 添加用户

```bash
cat /etc/passwd       # 查看用户
useradd xzh -g xzh -d /opt/xzh.webapp -s /sbin/nologin
passwd xzh
chmod -R 777 /opt/xzh.webapp
userdel xzh
```

#### 1.5.4 启动服务

```bash
systemctl start vsftpd.service      # 启动
systemctl enable vsftpd.service     # 开机自启
```

### 1.6 NFS

#### 1.6.1 安装

```bash
yum install -y nfs-utils
```

#### 1.6.2 创建共享目录

```bash
mkdir -p /share/nfs/software
chmod o+w /share/nfs/software   # 非root用户访问时需要增加其他组的写权限
touch {1..5}.txt    # 批量创建测试文件
```

#### 1.6.3 共享配置

```bash
vi /etc/exports 
# 添加
/share/nfs/software *(rw,no_root_squash) # *代表对所有IP都开放此目录，rw是读写
```

#### 1.6.4 启动服务

```bash
systemctl start nfs-server
systemctl enable nfs-server
```

#### 1.6.5 客户端测试

```bash
yum install -y nfs-utils.x86_64
mkdir -p /mnt/nfs/software      # 添加挂载文件夹
showmount -e 172.17.17.171      # 查看NFS共享目录
mount.nfs 172.17.17.171:/share/nfs/software /mnt/nfs/software   # 挂载目录
fuser -km /mnt/nfs/software     # kill 挂载进程
umount /mnt/nfs/software        # 取消目录
df -h
echo "172.17.17.171:/share/nfs/software /mnt/nfs/software nfs defaults 0 0" >> /etc/fstab    # 永久挂载
```

### 1.7 Samba

#### 1.7.1 安装

```bash
yum -y install samba
rpm -aq|grep ^samba
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

#### 1.7.2 用户默认访问家目录

1. 修改配置

```bash
vi /etc/samba/smb.conf
# 编辑
[homes]
    comment = Home Directories
    browseable = No
    writable = yes
```

2. 添加访问用户

```bash
useradd zhangsan        # 添加系统用户
smbpasswd -a zhangsan   # 添加smaba用户
pdbedit -L zhangsan     # 查看用户
pdbedit -a zhangsan     # 修改smaba用户密码
```

3. 启动服务

```bash
systemctl start smb
systemctl start nmb
```

4. 客户端测试

```bash
yum install -y samba-client cifs-utils
smbclient //172.17.17.201/zhangsan -U zhangsan
mkdir -p /mnt/samba/zhangsan
mount.cifs -o user=zhangsan,pass=123456 //172.17.17.201/zhangsan /mnt/samba/zhangsan
fuser -km /mnt/samba/zhangsan     # kill 挂载进程
umount /mnt/samba/zhangsan        # 取消目录
```

#### 1.7.3 匿名用户访问

1. 修改配置

```bash
mkdir -p /share/samba/anon
chmod o+w /share/samba/anon

vi /etc/samba/smb.conf
# 添加
[anon_share]
	path=/share/samba/anon
	public = yes
	writable = yes
```

2. 启动服务

```bash
systemctl restart smb
systemctl restart nmb
```


3. 客户端测试

```bash
mkdir -p /mnt/samba/software
smbclient //172.17.17.201/anon_share
mount.cifs -o user=zhangsan,pass=123456 //172.17.17.201/anon_share /mnt/samba/software
fuser -km /mnt/samba/software     # kill 挂载进程
umount /mnt/samba/software        # 取消目录
```

总结：
1. samba服务默认是基于用户名和密码认证的服务
2. samba服务的用户必须是samba服务器上存在的用户，密码必须是samba数据库里的密码
3. 对于发布的共享资源，默认情况下本地用户是可以访问的，匿名用户是否访问看是否打开public=yes

### 1.8 Telnet

#### 1.8.1 安装

```bash
yum -y install telnet-server xinetd
rpm -q xinetd telnet-server # 确认安装成功
```

#### 1.8.2 修改配置

```bash
cat /etc/xinetd.conf  | grep -v ^# | grep -v ^$

defaults
{
    log_type        = SYSLOG daemon info        # 日志类型，表示使用syslog进行服务登记。
    log_on_failure  = HOST                      # 失败日志，失败后记录客户机的IP地址。
    log_on_success  = PID HOST DURATION EXIT    # 成功日志，记录客户机的IP地址和进程ID
    cps             = 50 10                     # 表示每秒50个连接，如果超过限制，则等待10秒。主要用于对付拒绝服务攻击。
    instances       = 50        # 最大连接数
    per_source      = 10        # 每个IP地址最大连接数
    v6only          = no        # 不使用ipv6
    groups          = yes       # 确定该服务的进程组ID，/etc/group
    umask           = 002       # 文件生成码反掩码   666（664） 777(775)
}
includedir /etc/xinetd.d        # 外部调用的目录
```

#### 1.8.3 启动服务

```bash
systemctl start xinetd.service
systemctl enable xinetd.service

systemctl start telnet.socket
systemctl enable telnet.socket
```

?> 如果root用户默认无法登录，修改

```bash
vi /etc/securetty
# 添加到最后位置
pts/0 
pts/1
```

#### 1.8.4 客户端测试

```bash
yum install -y telnet
telnet 192.168.100.1 # 输入用户名和密码
```

### 1.9 时间同步服务器

#### 1.9.1 时间和时区设置

```bash
timedatectl                             # 查看时区
timedatectl list-timezones              # 查看所有可以使用的时区
timedatectl set-timezone Asia/Shanghai  # 设置本地时区
timedatectl set-timezone UTC            # 设置UTC时区

timedatectl set-local-rtc 1 # 硬件时钟协调本地时间
timedatectl set-local-rtc 0 # 硬件时钟协调世界时间
timedatectl status

# 时间设置（需要关闭NTP服务）
timedatectl set-ntp true
timedatectl set-ntp false

timedatectl set-time 16:40:30
timedatectl set-time 2022-07-04
timedatectl set-time "2022-07-04 16:44:30"
```


#### 1.9.2 NTP服务

1. 安装NTP

```bash
yum install -y ntp
rpm -ql ntp
```

2. 修改配置

```bash
vim /etc/ntp.conf
# 添加配置
restrict 172.17.17.0 mask 255.0.0.0 nomodify notrap	# 允许172.17.17.0/24网段的主机同步时间
```

3. 启动服务

```bash
service ntpd start
service enable start
netstat -tunlp|grep ntpd
```

4. 客户端测试

```bash
yum install -y  ntpdate
timedatectl set-ntp false   # 关闭时间同步
timedatectl set-time "2022-07-04 16:44:30" # 模拟时间错乱
ntpdate 172.17.17.201
```

#### 1.9.2 xinetd服务

1. 安装

```bash
yum -y install xinetd
```

2. 修改配置

```bash
vim /etc/xinetd.d/time-dgram
# 修改
disable = no

vim /etc/xinetd.d/time-stream
# 修改
disable = no
```

3. 启动服务

```bash
systemctl start xinetd
systemctl enable xinetd
netstat -ntlup |grep :37
```

4. 客户端测试

```bash
date -s "2022-07-04 16:44:30"
rdate -s 172.17.17.201
```

### 1.10 Rsyslog系统日志

#### 1.10.1 系统配置

1. 配置文件

```bash
systemctl status rsyslog
systemctl restart rsyslog

rpm -qc rsyslog
# 配置文件
/etc/logrotate.d/syslog # 日志轮转
/etc/rsyslog.conf       # 主配置文件
/etc/sysconfig/rsyslog

cp -p /etc/rsyslog.conf /etc/rsyslog.conf.bak   # 备份
cat /etc/rsyslog.conf.bak | grep -v "#" | grep -v "^$" > /etc/rsyslog.conf  # 去掉空行和注释
```

2. 日志格式：文本日志/二进制日志/数据库日志

```bash
/var/log/boot.log   # 系统引导日志，记录开机启动信息
/var/log/dmesg      # 核心的启动日志
/var/log/messages   # 系统的日志文件
/var/log/maillog    # 邮件服务的日志
/var/log/xferlog    # ftp服务的日志
/var/log/secure     # 网络连接及系统登录的安全信息
/var/log/cron       # 定时任务的日志
/var/log/wtmp       # 记录所有的登入和登出  last -f 查看
/var/log/btmp       # 记录失败的登入尝试
```

3. 日志级别

```lua
7 debug 调试信息的日志，日志信息最多
6 info 一般信息的日志，最常用
5 notice 最具有重要性的普通条件的信息
4 warning 警告级别
3 error 错误级别，阻止某个功能或者模块不能正常工作的信息
2 crit 严重级别，阻止整个系统或者整个软件不能正常工作的信息
1 alert 需要立刻修改的信息
0 emerg 内核崩溃等严重信息
```

4. 日志类型

```bash
auth    # 用户认证时产生的日志
authpriv    # ssh、ftp等登录信息的验证信息
cron    # 系统执行定时任务产生的日志。
daemon  # 一些守护进程产生的日志
kern    # 系统内核日志
lpr     # 打印相关活动
mail    # 邮件日志
mark    # 服务内部的信息，时间标识
news    # 网络新闻传输协议(nntp)产生的消息。
security
syslog  # 系统日志
user    # 用户进程
uucp    # Unix-to-Unix Copy 两个unix之间的相关通信
console # 针对系统控制台的消息。
ftp     # FTP产生的日志
local0~local7   # 自定义程序使用
```

#### 1.10.2 本地日志管理

1. 测试mail日志

```bash
vi /etc/rsyslog.conf    # 查看邮件日志保存目录

echo haha |mail -s "test mail log" zhangsan # 服务器向zhangsan发送测试邮件
tail -f /var/spool/mail/zhangsan            # 客户端查看接收到的邮件
```

2. 测试ssh日志

修改ssh默认日志载体

```bash
vim /etc/ssh/sshd_config
# 修改内容
SyslogFacility LOCAL6   # 

systemctl restart sshd  # 重启服务
```

修改ssh日志记录位置

```bash
vim /etc/rsyslog.conf
# 修改内容
*.info;mail.none;authpriv.none;cron.none;local6.none    /var/log/messages  # LOCAL6设备载体的日志不记录到messages中
local6.*    /var/log/ssh   # 指定LOCAL6设备载体的日志记录到指定位置

systemctl restart rsyslog  # 重启服务
```

#### 1.10.3 远程日志管理

1. 服务端配置

开启远程接收日志端口

```bash
vim /etc/rsyslog.conf   
# 修改内容
$ModLoad imudp  # 开启udp接收端口
$UDPServerRun 514

$ModLoad imtcp  # # 开启tcp接收端口
$InputTCPServerRun 514
```

重启服务

```bash
systemctl restart rsyslog
```

2. 客户端配置

修改ssh日志载体

```bash
vim /etc/ssh/sshd_config
# 修改内容
SyslogFacility LOCAL0   # 修改ssh默认日志载体

systemctl restart sshd  # 重启服务
```

修改rsyslog将日志发送到服务器

```bash
vim /etc/rsyslog.conf
# 修改内容
local0.*        @172.17.17.52:514   # @代表UDP协议传输；@@代表TCP协议传输

systemctl restart rsyslog  # 重启服务
```

3. 客户端测试

登录客户端ssh，查看日志服务端的输出

```bash
tail -f /var/log/messages   # 在日志服务端打开日志文件
```

4. 远程日志保存到指定的文件


```bash
vim /etc/rsyslog.conf   # 修改日志服务器配置
# 修改内容
$template DynFile,"/var/log/system-%HOSTNAME%.log"
local0.*	?DynFile

systemctl restart rsyslog  # 重启服务
ll /var/log/system*
```

客户端修改hostname以后必须重启rsyslog

#### 1.10.3 日志轮转

1. 配置文件

```bash
# 常用的指令解释，可以在man logrotate 查看。
daily                   # 指定转储周期为每天
monthly                 # 指定转储周期为每月
weekly                  # 每周轮转一次(monthly)
rotate 4                # 同一个文件最多轮转4次，4次之后就删除该文件
create 0664 root utmp   # 轮转之后创建新文件，权限是0664，属于root用户和utmp组
dateext                 # 用日期来做轮转之后的文件的后缀名
compress                # 用gzip对轮转后的日志进行压缩
minsize 30K             # 文件大于30K，而且周期到了，才会轮转
size 30k                # 文件必须大于30K才会轮转，而且文件只要大于30K就会轮转，不管周期是否已到
copytruncate            # 用于还在打开中的日志文件，把当前日志备份并截断
missingok               # 如果日志文件不存在，不报错
notifempty              # 如果日志文件是空的，不轮转
delaycompress           # 下一次轮转的时候才压缩
sharedscripts           # 不管有多少个文件待轮转，prerotate 和 postrotate 代码只执行一次
prerotate               # 如果符合轮转的条件，则在轮转之前执行prerotate和endscript 之间的shell代码
postrotate              # 轮转完后执行postrotate 和 endscript 之间的shell代码
```

```bash
logrotate -f /etc/logrotate.conf    # 强制轮转
```

2. nginx轮转

```bash
/usr/local/nginx/logs/*.log{
    create 0640 nginx root
    daily
    rotate 10
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /usr/local/nginx/logs/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

3. 自定义日志

将原日志文件复制一份，然后将原日志文件清空。这种截断方式不需要重启服务

```bash
mkdir -p /opt/tomcat/logs/backlog
cd /etc/logrotate.d/
vi tomcat8080
# 添加内容
/opt/tomcat/logs/catalina.out{
    copytruncate
    daily
    rotate 7
    su root root
    olddir backlog
    dateext dateformat %Y%m%d%s
}
```

### 1.11 Apache

#### 1.11.1 安装

```bash
yum install -y httpd
yum -y install httpd-manual.noarch	# 访问地址http://39.105.58.136/manual/
rpm -ql httpd
systemctl start httpd
```

#### 1.11.2 配置文件

```bash
/etc/httpd/conf/httpd.con       # 主配置文件
/etc/httpd/conf.d/*.conf        # 子配置文件
/etc/httpd/conf.d/welcome.conf  # 默认测试页面
/etc/httpd/logs                 # 日志目录 /var/log/httpd/ 硬链接
/etc/httpd/modules              # 库文件 /usr/lib64/httpd/modules 硬链接
/etc/httpd/run                  # pid信息
/etc/logrotate.d/httpd          # 轮转日志
/etc/sysconfig/httpd            # 额外配置文件
```

#### 1.11.3 共享文件

1. 软连接方式

```bash
vi /etc/httpd/conf/httpd.conf
# 编辑内容
DocumentRoot "/var/www/html"
<Directory "/var/www/html">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

mkdir -p /data
touch /data/file{1..5}

ln -s /data/ /var/www/html/share    # 创建链接
systemctl restart httpd             # 重启服务
http://39.105.58.136/share/         # 测试地址

```

2. 别名方式

```bash
vi /etc/httpd/conf.d/autoindex.conf
# 添加
Alias /test/ "/data/"
<Directory "/data">
    Options Indexes MultiViews FollowSymlinks
    AllowOverride None
    Require all granted
</Directory>

systemctl restart httpd             # 重启服务
http://39.105.58.136/test/          # 测试地址
```

#### 1.11.4 虚拟主机

```bash
vi /etc/httpd/conf/httpd.conf   # 添加监听端口
# 编辑
Listen 80
Listen 7070

vi /etc/httpd/conf.d/vhost.conf
# 添加
<VirtualHost *:7070>
        ServerName webmaster@dummy-host.example.com
        DocumentRoot "/data2/"
        DirectoryIndex index.php index.html
        <Directory "/data2">
                Options -Indexes +FollowSymlinks
                AllowOverride All
                Require all granted
        </Directory>
        ErrorLog "logs/lot_9xzb-error_log"
        CustomLog "logs/lot_9xzb-access_log" common
</VirtualHost>
```

### 1.12 iptables

#### 1.12.1 安装

```bash
yum install iptables-services

systemctl start iptables
systemctl enable iptables
systemctl status iptables
/etc/init.d/iptables stop

vi /etc/sysconfig/iptables
systemctl restart iptables
```

#### 1.12.2 filter

1. 处理动作

```bash
-s 192.168.134.0/24     # 源地址
-d 192.168.134.1        # 目标地址
-p tcp|upd|icmp         # 协议
-i lo                   # input 从lo接口进入的数据包
-o eth0                 # output 从eth0出去的数据包
-p tcp --dport 80       # 目标端口是80,必须和-p tcp|udp 连用
-p udp --dport 53       # 目标端口是53/udp
```

2. 语法示例

```bash
iptables -t filter -F                                   # 清空filter表的所有规则
iptables -t filter -L --line-numbers                    # 查看规则编号
iptables -t filter -A INPUT -s 10.1.1.3 -j ACCEPT       # 允许源地址为10.1.1.3进入
iptables -t filter -A INPUT ! -s 10.1.1.3 -j ACCEPT     # 不允许源地址为10.1.1.3进入
iptables -I INPUT -s 211.0.0.0/8 -j DROP                # 封整段
iptables -I INPUT -s 211.1.0.0/16 -j DROP               # 封IP段
iptables -I INPUT -s 61.37.81.0/24 -j DROP              # 封IP段

iptables -t filter -A INPUT -s 10.1.1.3 -j ACCEPT       # 允许源地址为10.1.1.3进入
iptables -t filter -A INPUT ! -s 10.1.1.3 -j ACCEPT     # 不允许源地址为10.1.1.3进入
iptables -t filter -A INPUT -s 10.1.1.3 -j DROP         # 拒绝源地址为10.1.1.3进入
iptables -t filter -A OUTPUT -d 10.1.1.3 -j DROP        # 丢弃到达目标地址为10.1.1.3的包
iptables -t filter -A OUTPUT ! -d 10.1.1.3 -j ACCEPT    # 丢弃到达目标地址为10.1.1.3的包
iptables -t filter -A INPUT -d 10.1.1.2 -j DROP         # 丢弃所有到目标地址10.1.1.2的包	
iptables -t filter -A OUTPUT -s 10.1.1.2 -j ACCEPT      # 源地址为10.1.1.2出去的包全部允许

iptables -A INPUT -s 10.1.1.2 -p tcp --dport 80 -j ACCEPT       # 只允许10.1.1.2 9090端口的访问
iptables -A INPUT -s 10.1.1.2 -p tcp --dport 20:21,2000:300 -j ACCEPT                   # 指定连续端口
iptables -t filter -I INPUT -s 10.1.1.2 -p tcp -m multiport --dports 22,80 -j ACCEPT    # 指定多个不连续端口
iptables -t filter -A INPUT -m iprange --src-range 10.1.1.2-10.1.1.5 -j ACCEPT          # 指定网段范围
```

#### 1.12.3 nat

1. 匹配规则

```bash
-j SNAT         # 源地址转换 POSTROUTING
-j DNAT         # 目标地址转换 PREROUTING
-j MASQUERADE   # 地址伪装
```

2. SNAT

```bash
iptables -t nat -A POSTROUTING -s 10.1.1.0/24 -j SNAT --to 2.2.2.1  # 内访外
iptables -t nat -A POSTROUTING -s 10.1.1.0/24 -j MASQUERADE
```

3. DNAT

```bash
iptables -t nat -A PREROUTING -d 2.2.2.1 -p tcp --dport 80 -j DNAT --to 10.1.1.3    # 外访内
```

#### 1.12.4 mangle

#### 1.12.5 raw

### 1.13 firewalld

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
firewall-cmd --reload                                    # 重载防火墙配置
```

### 1.14 puppet

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

shutdown -h now # 关机
shutdown -r now # 重启
```

### 2.2 文件

```bash
# 替换
ls -a
sed 's/6379/6380/g' redis-6379.conf > redis-6380.conf
echo 6379 6380 6381 16379 16380 16381 | xargs -t -n 1 cp /usr/local/redis/conf/redis.conf   # 文件批量拷贝至当前目录下的指定文件夹内

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

# 删除
find /doc -name '*bak' -exec rm {} \;               # 会从/doc目录开始往下找，找到凡是文件名结尾为 bak的文件，把它删除掉
find ./ -type f | xargs rm -rf;                     # 当前路径下文件类全部删除
find ./ -type f -delete;                            # 当前路径下文件类全部删除
find . -inum 105267648 -exec rm -i {} \;            # 通过inode号交互式删除文件
find ./ -inum 105267651 -delete                     
```

### 2.3 磁盘

1. 监控

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

### 2.4 网络

1. 监控

```bash
ps -aux | grep redis          # 查看启动进程参数
lsof -i:80                    # 可以看到pid和用户 
netstat -tunlp | grep 8080    # 查看端口进程号
netstat -anp | grep 17010pos  # 查看应用占用端口

traceroute -I www.163.com           # traceroute默认使用udp方式, 如果是-I则改成icmp方式
traceroute -M 3 www.163.com         # 从ttl第3跳跟踪
traceroute -p 8080 192.168.10.11    # 加上端口跟踪

route -n            # 查看路由,显示ip,不解析
route del default   # 删除默认路由
route add default gw 192.168.1.110      # 添加一个默认网关，把所有不知道的网络交给网关来转发
route del default gw 192.168.1.110      # 删除默认网关	        
route add -host 192.168.3.1 gw 192.168.1.110    # 对一个具体的ip添加路由
rouate add -net 192.168.2.0/24 dev eth0         # 对一个网络添加一个新的路由（另一个网段）
```

2. 端口检测

```bash
yum install nc

nc -z -w 3 192.168.20.183 7443 && echo ok || echo not ok
nc -v -w 10 -z 192.168.20.183 7443
nc -v -w 2 -z 127.0.0.1 7000-7500

yum install nmap
nmap www.baidu.com

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

### 2.5 编译

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

### 2.6 shell

#### 2.6.1 常用命令

```bash
# solr后台运行,并且有nohup.out输出
nohup java -Djetty.port=8080 -jar /opt/solr-4.7.2/example/start.jar &

# openfire后台运行, 不输出任何日志
nohup /opt/openfire/bin/openfire.sh >/dev/null &    

# sentinel后台运行, 并将错误信息做标准输出到日志中
nohup java -Dserver.port=9000 -jar sentinel-dashboard-1.7.2.jar >out.log 2>&1 & 
```


#### 2.6.2 tomcat

1. 启动

```bash
sh /data/tomcat_webapp_3001/bin/shutdown.sh
sleep 2s
ps -ef | grep tomcat_webapp_3001 | grep -v grep | awk '{print $2}'| xargs kill -9
sleep 1s
sh /data/tomcat_webapp_3001/bin/startup.sh;tail -f /data/tomcat_webapp_3001/logs/catalina.out
```

2. war部署

```bash
sh /opt/tomcat/bin/shutdown.sh
sleep 2s
ps -ef | grep /opt/tomcat/ | grep -v grep | awk '{print $2}'| xargs kill -9
sleep 1s
rm -rf /opt/tomcat/webapps/servlet*
cp -r /opt/tomcat/code/servlet-2.war /opt/tomcat/webapps/servlet.war
sh /opt/tomcat/bin/startup.sh;tail -f /opt/tomcat/logs/catalina.out
```

#### 2.6.3 Spring Boot

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

#### 2.6.4 xsync

```bash
yum install -y rsync
```

### 2.7 快捷键

```bash
ctrl + z / fg                       # 挂起
ctrl + s / q                        # 锁屏
ctrl + u                            # 前删除
ctrl + k                            # 后删除
Ctrl + Shift + c                    # 复制
Ctrl + Shift + v                    # 粘贴    

0  # 光标移到行首(数字0)
$  # 光标移至行尾
shift + g # 跳到文件最后
gg # 跳到文件头
```

## 3. 虚拟机

### 3.1 初始化

```bash
yum install -y zip unzip telnet lsof ntpdate openssh-server wget net-tools.x86_64
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
/usr/sbin/ntpdate ntp4.aliyun.com;/sbin/hwclock -w      # 同步时间

systemctl stop iptables.service
systemctl disable iptables.service  # 关闭
systemctl stop firewalld.service
systemctl disable firewalld.service # 关闭
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

### 3.2 ssh

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

### 3.3 网络配置

```bash
vi /etc/hosts
vi /etc/resolv.conf  nameserver 192.168.0.1    # 修改DNS
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3 # 修改IP
vi /etc/sysconfig/network                      # 修改网关 GATEWAY=192.168.3.1
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

### 3.4 更换yum源

```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo_bak  # 备份本地yum源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo  # 获取阿里yum源配置文件
yum clean all # 清空缓存  
yum makecache # 更新yum缓存
```

### 3.5 卸载软件

```bash
rpm -e --nodeps `rpm -qa | grep mariadb`
```

### 3.6 安装vim

```bash
yum -y install vim*
vi /etc/vimrc       # 添加 colorscheme murphy
vi /etc/profile     # 添加 alias vi=vim
source /etc/profile 
```

```bash
:set nu                 # 显示行号:set nonu
vim +3 /etc/passwd      # 定位到第三行
vim +/sssd /etc/passwd  # 定位到sssd所在的行
```

## 4. 开发环境

### 4.1 Java

#### 4.1.1 JDK

1. 在线安装

```bash
yum install java-1.8.0-openjdk
vim /etc/profile

export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export PATH=$PATH:$JAVA_HOME/bin
source /etc/profile   # 配置生效
```

2. 解压安装

```bash
cd /opt/software
tar -zxvf jdk-8u211-linux-x64.tar.gz
mv jdk1.8.0_211/ /usr/local/
vim /etc/profile

export JAVA_HOME=/usr/local/jdk1.8.0_211
export PATH=$PATH:$JAVA_HOME/bin
source /etc/profile   # 配置生效
```

#### 4.1.2 Maven

1. 上传解压

```bash
cd /opt/software
tar -xzf apache-maven-3.5.4-bin.tar.gz -C /opt    # 解压
```

2. 配置环境变量

```bash
vim /etc/profile
# 添加
export MAVEN_HOME=/opt/apache-maven-3.5.4
export MAVEN_OPTS="-Xms4096m -Xmx4096m"
export PATH=:$MAVEN_HOME/bin:$PATH

# 生效
source /etc/profile
mvn -v
```

3. 配置仓库

```bash
vim /opt/apache-maven-3.5.4/conf/settings.xml
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

### 4.3 TypeScript

```bash
npm init -y                     # 生成package.json配置文件
npm install -g typescript --registry=https://registry.npm.taobao.org # 全局安装ts，如果安装过了，忽略这个命令
tsc --init                      # 生成tsconfig.json配置文件
npm install rollup typescript rollup-plugin-typescript2 "@rollup/plugin-node-resolve" rollup-plugin-serve -D  # 安装rollup环境依赖
tsc -w                          # 手动编译
npm install ts-node -g --force  # 配合插件Code Runner
```

### 4.4 Golang

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