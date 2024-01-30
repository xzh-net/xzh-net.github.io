# CentOS Linux release 7.9.2009

- 官方网址：https://www.centos.org
- RPM资源：https://www.rpmfind.net

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

> 服务器的地址必须与仅主机模式中设置的ip网段相同

```bash
vim /etc/dhcp/dhcpd.conf

default-lease-time 259200;    # 预设租期3天
max-lease-time 518400;        # 最大租期6天
option domain-name "xuzhihao.net";         # 指定默认域名
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
zone "xuzhihao.net" IN {
        type master;
        file "xuzhihao.net.zone";
        allow-update { none; };
};
```

5. 配置zone文件

```bash
cp -p /var/named/named.localhost /var/named/xuzhihao.net.zone    # 一定要使用-p复制用户组和权限
vi /var/named/xuzhihao.net.zone

$TTL 1D
@       IN SOA  xuzhihao.net.  rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@       NS      dns1.xuzhihao.net.   # dns1表示命名空间，可以有多个，但是后面的Ａ记录保持一致就行
dns1    A       127.0.0.1           # 一定是当前DNS服务器的IP
www     A    192.168.3.200          # 根据实际填写
```

6. 检查配置文件的语法错误

```bash
named-checkconf /etc/named.conf
named-checkconf /etc/named.rfc1912.zones

cd /var/named/
named-checkzone xuzhihao.net.zone xuzhihao.net.zone # 区域文件写2遍
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

nslookup www.xuzhihao.net
nslookup -type=txt www.xuzhihao.net  # 验证txt值
dig @172.17.17.201 www.xuzhihao.net
host www.xuzhihao.net
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
@	IN SOA	xuzhihao.net. rname.invalid. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum
@	NS	dns2.xuzhihao.net.   # 如果dns2.xuzhihao.net在正向域文件中存在，可以不用写A记录
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

yum install bind-utils
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
zone "xuzhihao.net" IN {
        type slave;
        masters {172.17.17.201;};       # 指定master dns的ip地址
        file "slaves/xuzhihao.net.zone"; # 同步过来的文件的保存路径及名字
};
```

3. 修改dns服务器master节点

```bash
vi /etc/named.rfc1912.zones

# 修改以下内容：
zone "xuzhihao.net" IN {
        type master;
        file "xuzhihao.net.zone";
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
ll  # 可见同步过来的xuzhihao.net.zone文件
```

6. 客户端测试

```bash
echo nameserver 172.17.17.201 > /etc/resolv.conf  # 客户端机器添加dns服务器
echo nameserver 172.17.17.200 >> /etc/resolv.conf 

nslookup www.xuzhihao.net
dig @172.17.17.201 www.xuzhihao.net
host www.xuzhihao.net
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

# root用户下无法使用chattr
cp /usr/bin/chattr /usr/bin/chattr2
chmod 755 /usr/bin/chattr2
chattr2 -i /usr/bin/chattr
chmod 755 /usr/bin/chattr
ls -la /usr/bin/chattr  

lsattr authorized_keys      # 查看属性
chattr -ia authorized_keys  # 清理属性
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

```conf
#设定不允许匿名访问
anonymous_enable=NO
#设定本地用户可以访问注意：主要是为虚拟宿主用户，如果该项目设定为NO那么所有虚拟用户将无法访问
local_enable=YES
#设定可以进行写操作
write_enable=YES
#设定上传后文件的权限掩码
local_umask=022
#禁止匿名用户上传
#anon_upload_enable=NO
#禁止匿名用户建立目录
#anon_mkdir_write_enable=NO
#设定开启目录标语功能
dirmessage_enable=YES
#设定端口20进行数据连接
connect_from_port_20=YES
#设定禁止上传文件更改宿主
#chown_uploads=NO
#设定开启日志记录功能
xferlog_enable=YES
#设定日志使用标准的记录格式
xferlog_std_format=YES
#设定Vsftpd的服务日志保存路径
#	注意，该文件默认不存在必须要手动touch出来，并且由于这里更改了Vsftpd的服务宿主用户为手动建立的Vsftpd
#	必须注意给与该用户对日志的写入权限，否则服务将启动失败
#	仅在(xferlog_enable=YES并且xferlog_std_format=YES)或者(dual_log_enable=YES)时有意义。
xferlog_file=/var/log/vsftpd.log
#设定支撑Vsftpd服务的宿主用户为手动建立的Vsftpd用户
#	注意，一旦做出更改宿主用户后，必须注意一起与该服务相关的读写文件的读写赋权问题
#	比如日志文件就必须给与该用户写入权限等
#nopriv_user=vsftpd
#设定支持异步传输功能
#async_abor_enable=YES
#设定支持ASCII模式的上传功能
#ascii_upload_enable=YES
#设定支持ASCII模式的下载功能
#ascii_download_enable=YES
#设定Vsftpd的登陆标语
#ftpd_banner=WelcometoAweiFTPservers
#禁止本地用户登出自己的FTP主目录
chroot_local_user=YES
#设定启用虚拟用户功能
#guest_enable=YES
#设定PAM服务下Vsftpd的认证文件名(默认目录:/etc/pam.d/)
#	以下这些是关于Vsftpd虚拟用户支持的重要配置项目
#	默认Vsftpd.conf中不包含这些设定项目，需要自己手动添加配置
pam_service_name=vsftpd
#指定虚拟用户的宿主用户
#guest_username=vftp
#设定虚拟用户的权限符合他们的宿主用户
#virtual_use_local_privs=YES
#设定虚拟用户个人Vsftp的配置文件存放路径
#	也就是说，这个被指定的目录里，将存放每个Vsftp虚拟用户个性的配置文件，
#	一个需要注意的地方就是这些配置文件名必须和虚拟用户名相同
#user_config_dir=/etc/vsftpd/vconf
#是否以standalone模式运行。standalone模式是指vsftpd不由inetd启动，而是独自监听和处理连接
listen=YES
#standalone模式下,指定在哪个端口上监听(默认21)
listen_port=21
#standalone模式下,允许连接的最大客户端数量,默认"0"表示不限
max_clients=20000
#standalone模式下,同一个IP地址允许发起的最大并发连接数,默认"0"表示不限。
max_per_ip=20000
#客户端在两个FTP命令之间允许的最大间隔秒数。超时的客户端将被踢出。
idle_session_timeout=600
#含义与listen指令相同，只是在IPv6套接字上进行监听,该指令与listen是互斥的，不能同时使用
listen_ipv6=NO
#是否禁止userlist_file文件中的用户登陆。设为NO则根本无视该文件。
userlist_enable=YES
#是否通过tcp_wrappers对进入连接进行访问控制。仅在编译了tcp_wrappers支持的时候有效
#	tcp_wrappers可以针对每个单独的IP进行设置。
#	如果tcp_wrappers设置了VSFTPD_LOAD_CONF环境变量，那么vsftpd会话将会尝试加载其指定的配置文件
tcp_wrappers=YES
#是否允许使用PASV命令获得数据连接的地址端口对。
pasv_enable=yes
#PASV风格公网IP
pasv_address=39.105.58.136
#允许为PASV风格的数据连接使用的最小端口号,默认"0"表示不限
#vsftpd默认没有开启PASV模式，现在FTP只能通过PORT模式连接，要开启PASV默认需要通过下面的配置
pasv_min_port=50000
#允许为PASV风格的数据连接使用的最大端口号,默认"0"表示不限
pasv_max_port=50010
#解决无法登陆的问题
allow_writeable_chroot=YES
#使用户不能离开主目录
#chroot_list_enable=YES
#是否同时记录xferlog_file和vsftpd_log_file两份日志
dual_log_enable=YES
#指定vsftpd风格的日志文件的路径。
#	仅在(xferlog_enable=YES并且xferlog_std_format=NO)或者(dual_log_enable=YES)时有意义。
#	注意：如果syslog_enable=YES，那么日志将被发送到系统日志记录器，而不会记录到这里指定的文件中。
vsftpd_log_file=/var/log/vsftpd.log

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

> 如果root用户默认无法登录，修改

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

### 1.9 NTP

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

### 1.10 Rsyslog

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
/var/log/wtmp       # 记录所有的登入和登出  last -f 查看，或者使用 who -u /var/log/wtmp
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
firewall-cmd --zone=public --list-ports                  # 查看所有开启端口
firewall-cmd --query-port=6379/tcp                       # 查看端口是否开启
firewall-cmd --reload                                    # 重载防火墙配置
```

### 1.14 puppet

### 1.15 mailx

```bash
yum install mailx

vi /etc/mail.rc
set from=xcg992224@163.com
set smtp=smtps://smtp.163.com
set smtp-auth-user=xcg992224@163.com
set smtp-auth-password=xxxxxxxxxx
set smtp-auth=login
set ssl-verify=ignore
set nss-config-dir=/etc/pki/nssdb
```

```bash
echo "我们都是好孩子"|mailx -v -s "主题" xcg992224@163.com
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

shutdown -h now # 关机
shutdown -r now # 重启
```

![](../../assets/_images/deploy/centos7/resetpwd.png)

```bash
# 忘记root密码，开机狂按E，按图修改后按Ctrl+x重启输入
echo "123456" | passwd --stdin root
touch /.autorable
exec /sbin/init
```

### 2.2 文件

```bash
# 替换
sed 's/6379/6380/g' redis-6379.conf > redis-6380.conf
echo 6379 6380 6381 16379 16380 16381 | xargs -t -n 1 cp /usr/local/redis/conf/redis.conf   # 文件批量拷贝至当前目录下的指定文件夹内

# 压缩
zip -r xzh2021.zip * -x  './node_modules/*'         # 排除指定文件夹
tar -zcvf xzh2021.tar.gz  --exclude=node_modules *  # 排除指定文件夹
tar -zcvf jpg.tar *.jpg              # 打包所有图片
tar -zxvf file.tar -C /home/data/    # 解压到指定路径

# 拷贝复制
cp -f xxx.log                       # 复制并强制覆盖同名文件
mkdir -p /home/docker/data          # 级联创建目录
mkdir -p src/{test,main}/{java,resources}                               # 批量创建文件夹, 会在test,main下都创建java, resources文件夹
ln -s /usr/local/jdk1.8.0_202/bin/java /usr/bin/java                    # 创建软连接
scp -r code -P {port} root@192.168.3.201:/data/                                 # 远程复制
scp -r /usr/local/jdk1.8.0_202 root@192.168.3.201:/usr/local/jdk1.8.0_202       # 如果是全路径拷贝，源和目标路径需要一致
for i in {2..3}; do scp -r flink node$i:$PWD; done                              # 批量复制
sshpass -p "123456" scp -r /tmp/access.logs vjsp@192.168.3.120:/home            # 自动输入密钥

# Find查找
ll | grep hwcq.online               # 当前路径过滤显示
grep -n "hwcq.online" -r ./         # 当前路径递归向下查找内容
find / -name memcached              # 查找文件
find / -type f -size +100M          # 查找大文件 b/d/c/p/l/f 查是块设备、目录、字符设备、管道、符号链接、普通文件
find / -name 'conf' -type d         # 查找conf文件夹
find / -iname 'SerVer.xml'          # 查找server.xml文件的位置,忽略大小写
find / -name '*.mysql'              # 在目录下找后缀是.mysql的文件

# 删除
find /doc -name '*bak' -exec rm {} \;              # 会从/doc目录开始往下找，找到凡是文件名结尾为 bak的文件，把它删除掉
find . -type f | xargs rm -rf;                     # 当前路径下文件类全部删除
find . -type f -delete;                            # 当前路径下文件类全部删除
find . -inum 2891596 -exec rm -rf {} \;            # 通过inode号交互式删除文件
find . -inum 2891596 -delete

# 文件同步
yum install -y rsync
```

### 2.3 磁盘

#### 2.3.1 分区

```bash
fdisk -l
fdisk /dev/sdb
```

```lua
根据提示，依次输入n，p，空，空，空，wq，分区就开始了，很快就会完成。

步骤1，输入分区操作命令：n表示新建分区，即设定新的硬盘分割区。
步骤2，输入分区类型：p表示主分区，e表示扩展分区。
步骤3，输入分区号：1表示分区号，或直接回车表示分区号默认选择当前+1。
步骤4，输入起始扇区数，默认是从头开始即2048，一般都是选择默认。
步骤5，输入结束扇区数，默认是磁盘的尾部扇区，即整个磁盘的结束。如果需要创建多个分区，则结束分区要填写具体的扇区数，但是我们基本不这么做，一般我们要建的分区都是固定的大小，可以起始扇区选择默认，结束扇区输入+20G，则该分区为20GB，系统会自动选择结束扇区。
步骤6，保存退出：wq表示保存退出。
```

```bash
mkfs -t ext4 /dev/sdb1  # 格式化
mkdir /data
mount /dev/sdb1 /data   # 单独挂载到某个目录
df -lh                  # 查看磁盘使用情况和挂载情况
umount /dev/sdb1        # 卸载分区

echo '/dev/sdb1 /data ext4 defaults 0 0' >> /etc/fstab  # 自动挂载
cat /etc/fstab          # 查看写入分区信息
```

#### 2.3.2 FIO

FIO 是一个多线程IO生成工具，可以生成多种IO模式（随机、顺序、读、写四大类），用来测试磁盘设备的性能

```bash
yum install fio -y
```

```bash
# 4k顺序读
fio -filename=/tmp/fiotest  -direct=1 -iodepth 1 -thread -rw=read -rwmixread=70 -ioengine=psync -bs=4k -size=10G -numjobs=20 -runtime=60 -group_reporting -name=sqe_100read_4k >> fio.report
# 4k顺序写
fio -filename=/tmp/fiotest  -direct=1 -iodepth 1 -thread -rw=write -rwmixread=70 -ioengine=psync -bs=4k -size=10G -numjobs=20 -runtime=60 -group_reporting -name=sqe_100write_4k
# 4k随机读
fio -filename=/tmp/fiotest  -direct=1 -iodepth 1 -thread -rw=randread -rwmixread=70 -ioengine=psync -bs=4k -size=10G -numjobs=20 -runtime=60 -group_reporting -name=rand_100read_4k
# 4k随机写
fio -filename=/tmp/fiotest  -direct=1 -iodepth 1 -thread -rw=randwrite -rwmixread=70 -ioengine=psync -bs=4k -size=10G -numjobs=20 -runtime=60 -group_reporting -name=rand_100write_4k
# 4k顺序混合读写
fio -filename=/tmp/fiotest  -direct=1 -iodepth 1 -thread -rw=rw -rwmixread=70 -ioengine=psync -bs=4k -size=10G -numjobs=20 -runtime=60 -group_reporting -name=sqe_70read_4k
# 4k随机混合读写
fio -filename=/tmp/fiotest  -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=4k -size=10G -numjobs=20 -runtime=60 -group_reporting -name=rand_70read_4k
```


### 2.4 网络

#### 2.4.1 route

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

#### 2.4.2 netcat 

Netcat 是Linux系统中的网络工具，其通过TCP和UDP协议在网络中读写数据

```lua
-k  在当前连接结束后保持继续监听
-l  用作端口监听，而不是发送数据
-n  不使用 DNS 解析
-N  在遇到 EOF 时关闭网络连接
-p  指定源端口
-u  使用 UDP 协议传输
-v  (Verbose)显示更多的详细信息
-w  指定连接超时时间
-z  不发送数据
```

1. 安装

```bash
yum install -y nc           # 安装
nc -lk 44444                # 开启监听服务
nc 192.168.2.100 44444      # 客户端连接
```

```bash
yum install nc --downloadonly --downloaddir=/opt/
yum localinstall *.rpm
```

2. 端口检测

```bash
nc -z -u -v 192.168.2.1 68      # 检测udp端口
nc -z -v 192.168.2.100 22       # 检测tcp端口
nc -z -w 3 192.168.20.183 7443 && echo ok || echo not ok    # 用于脚本返回值
```

3. 文件传输

```bash
nc -l 9995 >oauth_center_bak.tar            # 接收机器开启9995接收流文件到oauth_center_bak.tar
nc 192.168.2.100 9995 < oauth_center.tar    # 传输机器通过9555上传
```

4. 测试网速

```bash
nc -l 9991 >/dev/null                       # 接收机器开启9991将文件写入/dev/null
nc 192.168.2.100 9991 </dev/zero            # 发送方把无限个0发送给A机器的9991端口
```

验证

```bash
yum -y install dstat
dstat
```

#### 2.4.3 nmap 

Network Mapper是一款开源免费的针对大型网络的端口扫描工具，nmap可以检测目标主机是否在线、主机端口开放情况、检测主机运行的服务类型及版本信息、检测操作系统与设备类型等信息

```bash
yum install -y nmap
nmap www.baidu.com
```

#### 2.4.4 tcpdump

```bash
yum install tcpdump

tcpdump -n -X -i any port 445 -A    # 指定端口
tcpdump -i em4                      # 指定网卡
tcpdump -i em4 -nn 'src host 192.168.2.3'   # 监听来源ip
tcpdump -i em4 -nn 'dst host 192.168.2.3'   # 监听返回ip
```

#### 2.4.5 ifstat

ifstat是一个统计网络接口活动状态的工具

```bash
wget http://gael.roualland.free.fr/ifstat/ifstat-1.1.tar.gz # 下载
tar -zxvf ifstat-1.1.tar.gz
cd ifstat-1.1
./configure            # 默认会安装到/usr/local/bin/目录中
make && make install
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

#### 2.4.6 dstat

dstat是一个通用的系统资源统计工具，stat命令是一个用来替换vmstat、iostat、netstat、nfsstat和ifstat这些命令，是一个全能系统信息统计工具

```lua
-c：显示CPU系统占用，用户占用，空闲，等待，中断，软件中断等信息。
-C：当有多个CPU时候，此参数可按需分别显示cpu状态，例：-C 0,1 是显示cpu0和cpu1的信息。
-d：显示磁盘读写数据大小。
-D hda,total：include hda and total。
-n：显示网络状态。
-N eth1,total：有多块网卡时，指定要显示的网卡。
-l：显示系统负载情况。
-m：显示内存使用情况。
-g：显示页面使用情况。
-p：显示进程状态。
-s：显示交换分区使用情况。
-S：类似D/N。
-r：I/O请求情况。
-y：系统状态。
--ipc：显示ipc消息队列，信号等信息。
--socket：用来显示tcp udp端口状态。
-a：此为默认选项，等同于-cdngy。
-v：等同于 -pmgdsc -D total。
--output 文件：此选项也比较有用，可以把状态信息以csv的格式重定向到指定的文件中，以便日后查看。例：dstat --output /root/dstat.csv & 此时让程序默默的在后台运行并把结果输出到/root/dstat.csv文件中。
```

```bash
yum -y install dstat
```

```bash
dstat -t --top-cpu-adv 2 6          # 查看当前最耗CPU的进程名、PID和CPU占比以及读写信息
dstat -t --top-bio-adv 2 6          # 显示最高磁盘IO的进程
dstat -t --top-mem 2 6              # 查看当前最耗内存的进程
dstat -t  -dD sda,total  2 6        # 监控磁盘sda的读写状态
dstat -t  -n -N ens33,total 2 6     # 监控网卡的流量
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

#### 2.6.1 Tomcat

1. 一键启动

```bash
vi run_tomcat_web.sh
```

```bash
#!/bin/bash
appname=tomcat_web_8080
PIDS=$(ps -ef|grep ${appname} | grep -v grep | awk '{print $2}')

start(){
    /home/${appname}/bin/startup.sh
    echo "${appname} started"
}

echo " $PIDS "

for PID in $PIDS
    do kill -9 $PID
    echo " $PID has been killed"
done

start

ps -aux |grep ${appname}
```

替换复制脚本

```bash
sed 's/tomcat_web/tomcat_mobile/g' run_tomcat_web.sh > run_tomcat_mobile.sh
```

2. 虚拟主机

```conf
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>
  <Service name="Catalina">
	<Connector port="8080" protocol="org.apache.coyote.http11.Http11Nio2Protocol"
		   connectionTimeout="30000"
		   URIEncoding="UTF-8"
		   minSpareThreads="100"
		   maxThreads="4096"
		   acceptCount="10000"
		   enableLookups="false"   
		   disableUploadTimeout="true"
		   redirectPort="8443" />
    <Engine name="Catalina" defaultHost="localhost">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
				<Context path="/" docBase="/data/"  reloadable="true" privileged="true" sessionCookieName="web"/>
				<Context path="/mobile" docBase="/data/mobile"  reloadable="true" privileged="true" sessionCookieName="mobile"/>
      </Host>
	  <Host name="www.xzh.com"  appBase="webapps1"
            unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
				<Context path="/" docBase=""  reloadable="true" privileged="true" sessionCookieName="web"/>
				<Context path="/mobile" docBase="mobile"  reloadable="true" privileged="true" sessionCookieName="mobile"/>
      </Host>
    </Engine>
  </Service>
</Server>
```

Nginx修改对应配置

```conf
server {
	listen 80;
	server_name www.xuzhihao.net;
	location / {
		proxy_pass http://192.168.2.200:8080;
	}
}

server {
	listen 80;
	server_name node.xuzhihao.net;
	location / {
		proxy_pass http://www.xzh.com:8080;
	}
}
```

```bash
vi /etc/hosts
192.168.2.200 www.xzh.com
```

#### 2.6.2 Java

```bash
#!/bin/bash
appname=user-center.jar
PIDS=$(ps -ef|grep ${appname} | grep -v grep | awk '{print $2}')

start(){
    JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
    nohup java ${JAVA_OPT} -jar ${appname} > nohup.out 2>&1 &
    echo "${appname} started"
}

echo "$PIDS"

for PID in $PIDS
    do kill -9 $PID
    echo "$PID has been killed"
done

start

ps -aux |grep ${appname}
```

#### 2.6.3 RocketMQ

```bash
# MQ默认以类路径文件启动，其他环境下也可以打开注释使用jar文件启动
nohup sh bin/runserver.sh org.apache.rocketmq.namesrv.NamesrvStartup &
```

```bash
#!/bin/sh
#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

find_java_home()
{
    case "`uname`" in
        Darwin)
            JAVA_HOME=$(/usr/libexec/java_home)
        ;;
        *)
            JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
        ;;
    esac
}

find_java_home

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
#export SERVER="mq-server"
export CLASSPATH=.:${BASE_DIR}/conf:${BASE_DIR}/lib/*:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
# The RAMDisk initializing size in MB on Darwin OS for gc-log
DIR_SIZE_IN_MB=600

choose_gc_log_directory()
{
    case "`uname`" in
        Darwin)
            if [ ! -d "/Volumes/RAMDisk" ]; then
                # create ram disk on Darwin systems as gc-log directory
                DEV=`hdiutil attach -nomount ram://$((2 * 1024 * DIR_SIZE_IN_MB))` > /dev/null
                diskutil eraseVolume HFS+ RAMDisk ${DEV} > /dev/null
                echo "Create RAMDisk /Volumes/RAMDisk for gc logging on Darwin OS."
            fi
            GC_LOG_DIR="/Volumes/RAMDisk"
        ;;
        *)
            # check if /dev/shm exists on other systems
            if [ -d "/dev/shm" ]; then
                GC_LOG_DIR="/dev/shm"
            else
                GC_LOG_DIR=${BASE_DIR}
            fi
        ;;
    esac
}

choose_gc_options()
{
    # Example of JAVA_MAJOR_VERSION value : '1', '9', '10', '11', ...
    # '1' means releases befor Java 9
    JAVA_MAJOR_VERSION=$("$JAVA" -version 2>&1 | sed -r -n 's/.* version "([0-9]*).*$/\1/p')
    if [ -z "$JAVA_MAJOR_VERSION" ] || [ "$JAVA_MAJOR_VERSION" -lt "9" ] ; then
      JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
      JAVA_OPT="${JAVA_OPT} -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8 -XX:-UseParNewGC"
      JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:${GC_LOG_DIR}/rmq_srv_gc_%p_%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps"
      JAVA_OPT="${JAVA_OPT} -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m"
    else
      JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
      JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0"
      JAVA_OPT="${JAVA_OPT} -Xlog:gc*:file=${GC_LOG_DIR}/rmq_srv_gc_%p_%t.log:time,tags:filecount=5,filesize=30M"
    fi
}

choose_gc_log_directory
choose_gc_options
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages"
#JAVA_OPT="${JAVA_OPT} -jar ${BASE_DIR}/target/${SERVER}.jar"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

$JAVA ${JAVA_OPT} $@
```

## 3. 虚拟机设置

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

### 3.2 ssh配置

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
hostnamectl set-hostname xzh                   # 修改主机名
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

## 4. 环境搭建

### 4.1 Java Sdk

jdk 1.8

```bash
# Oracle最后一个商用免费版本
cd /opt/software
tar -zxvf jdk-8u202-linux-x64.tar.gz -C /usr/local/
vim /etc/profile

export JAVA_HOME=/usr/local/jdk1.8.0_202
export PATH=$PATH:$JAVA_HOME/bin
source /etc/profile   # 配置生效

# 免密分发
scp /etc/profile root@node02:/etc/
scp -r /usr/local/jdk1.8.0_202  root@node02:/usr/local/
```

### 4.2 Maven

1. 上传解压

```bash
cd /opt/software
tar -xzf apache-maven-3.6.3-bin.tar.gz -C /opt    # 解压
```

2. 配置环境变量

```bash
vim /etc/profile
# 添加
export MAVEN_HOME=/opt/apache-maven-3.6.3
export MAVEN_OPTS="-Xms4096m -Xmx4096m"
export PATH=$PATH:$MAVEN_HOME/bin

# 生效
source /etc/profile
mvn -v
```

3. 配置仓库

```bash
vim /opt/apache-maven-3.6.3/conf/settings.xml
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


### 4.3 Node

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

1. 配置环境

```bash
wget https://dl.google.com/go/go1.13.4.linux-amd64.tar.gz
tar zxvf go1.13.4.linux-amd64.tar.gz -C /usr/local/
vi /etc/profile
```

```conf
export GOROOT=/usr/local/go                     # Golang安装目录
export GOPATH=/opt/gocode                       # Golang项目代码目录
export GOBIN=$GOPATH/bin                        # go install后生成的可执行命令存放路径
export PATH=$PATH:$GOROOT/bin                   # 全局配置
export GO111MODULE=on                           # 模块支持
export GOPROXY=https://goproxy.cn               # 模块代理
export http_proxy=http://172.17.17.165:7890     # 翻墙
export https_proxy=http://172.17.17.165:7890
```

```bash
source /etc/profile
go version
go env
```

2. 创建第一个应用程序

```bash
cd /opt/gocode
mkdir markdown && cd markdown       # 创建项目文件夹
go mod init markdown-renderer       # 初始项目
vi markdown_trest.go
```

```go
package main

import "fmt"

func init() {
	fmt.Println("初始化成功")
}
func main() {
	fmt.Println("markdown渲染器启动成功")
}
```

```bash
go run markdown_trest.go
go build
go install
```

### 4.6 Python

```bash
#!/bin/bash
nohup python -m SimpleHTTPServer 8887 &
```