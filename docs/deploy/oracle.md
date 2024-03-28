# Oracle 11g R2

## 1. 安装

### 1.1 静默11.2.0.1.0

#### 1.1.1 安装依赖

```bash
yum -y install gcc gcc-c++ make binutils compat-libstdc++-33 elfutils-libelf elfutils-libelf-devel elfutils-libelf-devel-static glibc glibc-common glibc-devel ksh libaio libaio-devel libgcc libstdc++ libstdc++-devel numactl-devel sysstat unixODBC unixODBC-devel kernelheaders pdksh pcre-devel readline rlwrap
```

#### 1.1.2 创建用户和组

```bash
groupadd oinstall && groupadd dba && useradd -g oinstall -G dba oracle
passwd oracle
```

#### 1.1.3 创建安装包存放目录

```bash
mkdir -p /u01/software # 上传安装文件
```

#### 1.1.4 解压安装包

```bash
cd /u01/software
unzip linux.x64_11gR2_database_1of2.zip && unzip linux.x64_11gR2_database_2of2.zip
```

#### 1.1.5 创建oracle目录，并授权文件夹目录权限给oracle

```bash
mkdir -p /u01/app/oracle/product/11.2.0/dbhome_1
mkdir /u01/app/oracle/{oradata,inventory,fast_recovery_area,oradmp}
chown -R oracle:oinstall /u01/app/oracle/
chmod -R 775 /u01/app/oracle
```

#### 1.1.6 修改内核参数

kernel.shmmax官方建议值：
- 32位linux系统：可取最大值为4GB（4294967296bytes）-1byte，即4294967295。建议值为多于内存的一半，所以如果是32为系统，一般可取值为4294967295。32位系统对SGA大小有限制，所以SGA肯定可以包含在单个共享内存段中。
- 64位linux系统：可取的最大值为物理内存值-1byte，建议值为多于物理内存的一半，一般取值大于SGA_MAX_SIZE即可，可以取物理内存-1byte。内存为32G时，该值为`34359738367`（32x1024x1024x1024-1）

```bash
vim /etc/sysctl.conf
```

```conf
fs.aio-max-nr=1048576
fs.file-max=6815744
kernel.shmall=2097152
kernel.shmmni=4096
kernel.shmmax = 34359738367
kernel.sem=250 32000 100 128
net.ipv4.ip_local_port_range=9000 65500
net.core.rmem_default=262144
net.core.rmem_max=4194304
net.core.wmem_default=262144
net.core.wmem_max=1048586
```

使内核新配置生效
```bash
sysctl -p
```

#### 1.1.7 修改用户限制

vim /etc/security/limits.conf
```bash
oracle	soft	nproc	2047
oracle	hard	nproc	16384
oracle	soft	nofile	1024
oracle	hard	nofile	65536
oracle	soft	stack	10240
```

#### 1.1.8  修改/etc/pam.d/login文件

vim /etc/pam.d/login
```bash
# 找到这一行：        
session required pam_namespace.so 
# 在其下一行添加： 
session required /lib64/security/pam_limits.so
session required pam_limits.so
```

#### 1.1.9 修改/etc/profile文件

vim /etc/profile
```bash
# 添加以下内容
if [ $USER = "oracle" ]; then
  if [ $SHELL = "/bin/ksh" ]; then
    ulimit -p 16384
    ulimit -n 65536
  else
    ulimit -u 16384 -n 65536
  fi
fi
```

#### 1.1.10 设置oracle用户环境变量

```bash
su - oracle
vim .bash_profile
```
```bash
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1
export ORACLE_SID=orcl
export ORACLE_UNQNAME=$ORACLE_SID
export PATH=$ORACLE_HOME/bin:$PATH
export NLS_LANG=american_america.AL32UTF8
```
```bash
# 生效
source .bash_profile
```

#### 1.1.11 添加主机名与ip对应记录

```bash
su - root 
hostnamectl set-hostname oracledb 
vim /etc/hosts
127.0.0.1 oracledb
```

#### 1.1.12 编辑静默安装响应文件

```bash
su - oracle
cp -R /u01/software/database/response/ . && cd response/
vim db_install.rsp
```

```bash
# 设置以下内容
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=oracledb # 主机名
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oracle/inventory
SELECTED_LANGUAGES=en,zh_CN
ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=dba
DECLINE_SECURITY_UPDATES=true
```

#### 1.1.13 执行安装

```bash
cd /u01/software/database/
./runInstaller -silent -responseFile /home/oracle/response/db_install.rsp -ignorePrereq
```

```lua
[oracle@oracledb database]$ ./runInstaller -silent -responseFile /home/oracle/response/db_install.rsp -ignorePrereq
正在启动 Oracle Universal Installer...

检查临时空间: 必须大于 120 MB。   实际为 44483 MB    通过
检查交换空间: 必须大于 150 MB。   实际为 8063 MB    通过
准备从以下地址启动 Oracle Universal Installer /tmp/OraInstall2023-12-07_04-57-24PM. 请稍候...[oracle@oracledb database]$ [WARNING] [INS-32055] 主产品清单位于 Oracle 基目录中。
   原因: 主产品清单位于 Oracle 基目录中。
   操作: Oracle 建议将此主产品清单放置在 Oracle 基目录之外的位置中。
可以在以下位置找到本次安装会话的日志:
 /u01/app/oracle/inventory/logs/installActions2023-12-07_04-57-24PM.log
Oracle Database 11g 的 安装 已成功。
请查看 '/u01/app/oracle/inventory/logs/silentInstall2023-12-07_04-57-24PM.log' 以获取详细资料。

以 root 用户的身份执行以下脚本:
	1. /u01/app/oracle/inventory/orainstRoot.sh
	2. /u01/app/oracle/product/11.2.0/dbhome_1/root.sh


Successfully Setup Software.
```

> 等待...[WARING]可暂时忽略，此时安装程序仍在后台进行，如果出现[FATAL]，则安装程序已经异常停止了,当出现 Successfully Setup Software. 证明已经安装成功，然后根据提示操作


```bash
su - root
sh /u01/app/oracle/inventory/orainstRoot.sh
sh /u01/app/oracle/product/11.2.0/dbhome_1/root.sh
```

#### 1.1.14 以静默方式配置监听

```bash
su - oracle
source .bash_profile
netca /silent /responsefile /home/oracle/response/netca.rsp
```

```lua
[oracle@oracledb ~]$ netca /silent /responsefile /home/oracle/response/netca.rsp

正在对命令行参数进行语法分析:
参数"silent" = true
参数"responsefile" = /home/oracle/response/netca.rsp
完成对命令行参数进行语法分析。
Oracle Net Services 配置:
完成概要文件配置。
Oracle Net 监听程序启动:
    正在运行监听程序控制: 
      /u01/app/oracle/product/11.2.0/dbhome_1/bin/lsnrctl start LISTENER
    监听程序控制完成。
    监听程序已成功启动。
监听程序配置完成。
成功完成 Oracle Net Services 配置。退出代码是0
```

> 成功运行后，在 `/u01/app/oracle/product/11.2.0/dbhome_1/network/admin/` 中生成`listener.ora`和`sqlnet.ora`

```bash
cat $ORACLE_HOME/network/admin/listener.ora     # 查看监听器配置文件
cat $ORACLE_HOME/network/admin/sqlnet.ora       # 查看监听服务名配置文件
yum install net-tools
netstat -tunlp|grep 1521
```


#### 1.1.15 以静默方式建立新库，同时也建立一个对应的实例

```bash
su - oracle
egrep -v "(^#|^$)" /home/oracle/response/dbca.rsp       # 查看建库相应文件配置信息
vim /home/oracle/response/dbca.rsp      # 编辑应答文件

# 设置CREATEDATABASE以下参数
GDBNAME = "orcl"
SID = "orcl"
DATAFILEDESTINATION =/u01/app/oracle/oradata
RECOVERYAREADESTINATION=/u01/app/oracle/fast_recovery_area
CHARACTERSET = "AL32UTF8"
TOTALMEMORY = "4096"    # 物理内存的80%
```

静默配置
```bash
dbca -silent -responseFile /home/oracle/response/dbca.rsp
```

> 执行完后会先清屏，清屏之后没有提示，直接输入oracle用户的密码，回车，再输入一次，再回车。稍等一会，会开始自动创建，建库后进行实例进程检查 

```bash
ps -ef | grep ora_ | grep -v grep
```

#### 1.1.16 登录数据库

```bash 
su - oracle
lsnrctl status
lsnrctl start
lsnrctl stop
sqlplus / as sysdba
startup
shutdown immediate
select status from v$instance;
exit
```

#### 1.1.17 开机启动

1. 编辑dbstart和dbstut，有的版本dbstut找不到，可以忽略

```bash
su - oracle
vim $ORACLE_HOME/bin/dbstart
# 将
ORACLE_HOME_LISTNER =$1
# 改成
ORACLE_HOME_LISTNER=$ORACLE_HOME
```

2. 编辑/etc/oratab文件

```bash
vi /etc/oratab
# 将
orcl:/u01/app/oracle/product/11.2.0/dbhome_1:N
# 改成
orcl:/u01/app/oracle/product/11.2.0/dbhome_1:Y
```

3. 编辑/etc/rc.d/rc.local文件

切换为root用户，在/etc/rc.d/rc.local文件结尾添加以下内容

```bash
su - root
vim /etc/rc.d/rc.local 
```

```bash
su - oracle -lc "/u01/app/oracle/product/11.2.0/dbhome_1/bin/lsnrctl start"
su - oracle -lc "/u01/app/oracle/product/11.2.0/dbhome_1/bin/dbstart"
```

增加文件执行权限

```bash
chmod +x /etc/rc.d/rc.local
```


### 1.2 图形化11.2.0.4.0

#### 1.2.1 一键安装和配置VNC图形化相关

下载脚本：https://github.com/xzh-net/other/tree/main/oracle11g

```bash
#!/bin/bash
#因为脚本内容大量中文，临时设置中文环境
export LC_ALL=zh_CN.UTF-8
echo '------欢迎使用 CentOS7 Oracle 11G安装助手------'
echo '脚本使用环境如下：'
echo '操作系统: CentOS Linux release 7.9.2009 (Core)'
echo 'Oracle: linux.x64_11g_11.2.0.4'
echo '* 创建oracle用户和组。'
echo '* 搭建图形化的操作环境：VNC远程。'
echo '* 防火墙放行VNC端口5901和Oracle默认端口1521。'
echo '* 安装oracle安装程序依赖程序包。'
echo '* 安装中文字体解决中文乱码问题。'
echo '* 单独安装pdksh-5.2.14'
echo '------欢迎使用 CentOS7 Oracle 11G安装助手------'
echo '当前操作系统版本是：'
cat /etc/redhat-release
read -r -p "确定继续执行吗? [Y/n] " input

case $input in
    [yY][eE][sS]|[yY]|[1])
		echo '-------------正在安装VNC图形化相关软件----------------'
    #图形界面必备`X Window System`
    yum -y groupinstall "X Window System"
    #安装epel源
    yum -y install epel-release
    #安装VNC+图形需要的软件
    yum -y install tigervnc-server openbox xfce4-terminal tint2 cjkuni-ukai-fonts network-manager-applet
    echo '-------------自动修改/etc/xdg/openbox/autostart配置文件-------------'
    #自动修改/etc/xdg/openbox/autostart配置文件
    echo 'if which dbus-launch >/dev/null && test -z "$DBUS_SESSION_BUS_ADDRESS"; then' > /etc/xdg/openbox/autostart
    echo '       eval `dbus-launch --sh-syntax --exit-with-session`' >> /etc/xdg/openbox/autostart
    echo 'fi' >> /etc/xdg/openbox/autostart
    echo 'tint2 &' >> /etc/xdg/openbox/autostart
    echo 'nm-applet  &' >> /etc/xdg/openbox/autostart
    echo 'xfce4-terminal &' >> /etc/xdg/openbox/autostart
    echo ' ' >> /etc/xdg/openbox/autostart
    cat /etc/xdg/openbox/autostart
    echo '-------------防火墙放行VNC端口5901-------------'
    firewall-cmd --add-port=5901/tcp
    firewall-cmd --add-port=5901/tcp --permanent
    echo '-------------防火墙放行VNC端口1521-------------'
    firewall-cmd --add-port=1521/tcp
    firewall-cmd --add-port=1521/tcp --permanent
    echo '-------------安装oracle依赖的程序包-------------'
    yum -y install binutils compat-libcap1  compat-libstdc++-33 compat-libstdc++-33*.i686 elfutils-libelf-devel gcc gcc-c++ glibc*.i686 glibc glibc-devel glibc-devel*.i686 ksh libgcc*.i686 libgcc libstdc++ libstdc++*.i686 libstdc++-devel libstdc++-devel*.i686 libaio libaio*.i686 libaio-devel libaio-devel*.i686 make sysstat unixODBC unixODBC*.i686 unixODBC-devel unixODBC-devel*.i686 libXp
    echo '-------------正在安装中易宋体18030-------------'
    #新建字体目录
    mkdir -p /usr/share/fonts/zh_CN/TrueType
    #复制字体到字体目录
    cp zysong.ttf /usr/share/fonts/zh_CN/TrueType/zysong.ttf
    #更改字体文件属性，使其生效
    chmod 75 /usr/share/fonts/zh_CN/TrueType/zysong.ttf
    echo '字体安装完成！'
    ls /usr/share/fonts/zh_CN/TrueType
    echo '-------------正在尝试安装pdksh-5.2.14-------------'
    echo '尝试卸载冲突ksh-20120801-142.el7.x86_64'
    rpm -e ksh-20120801-142.el7.x86_64
    rpm -e ksh-20120801-143.el7_9.x86_64
    rpm -ivh pdksh-5.2.14-37.el5_8.1.x86_64.rpm
    echo '查询是否安装成功：'
    rpm -q pdksh
    echo '脚本执行完成！开始oracle安装吧!'
		;;

    [nN][oO]|[nN]|[0])
		echo "No"
       	;;

    *)
		echo "Invalid input..."
		exit 1
		;;
esac
```

#### 1.2.2 创建用户和组

```bash
groupadd oinstall && groupadd dba && useradd -g oinstall -G dba oracle
passwd oracle
```

#### 1.2.3 开启VNC服务

```bash
su oracle
# 首次运行，生成~/.vnc/xstartup等配置文件
vncserver :1 -geometry 1024x768
# 配置VNC默认启动openbox
echo "openbox-session &" > ~/.vnc/xstartup
# 停止服务
vncserver -kill :1
# 重新开启vnc服务
vncserver :1 -geometry 1024x768
```

使用 [VNC Viewer点击下载](https://www.realvnc.com/en/connect/download/viewer/) 连接192.168.3.200:5901


#### 1.2.4 上传并解压安装包

```bash
su oracle
unzip p13390677_112040_Linux-x86-64_1of7.zip
unzip p13390677_112040_Linux-x86-64_2of7.zip
```

#### 1.2.5 oracle用户登录vnc远程桌面

```bash
cd ~/database/
./runInstaller
```

不接收邮件通知更新

![](../../assets/_images/deploy/oracle/step1.png)

跳过软件更新

![](../../assets/_images/deploy/oracle/step2.png)

创建和配置数据库

![](../../assets/_images/deploy/oracle/step3.png)

选择服务器类

![](../../assets/_images/deploy/oracle/step4.png)

默认选择单实例数据库

![](../../assets/_images/deploy/oracle/step5.png)

默认典型安装

![](../../assets/_images/deploy/oracle/step6.png)

设置实例和密码，其他默认即可。这里密码要在大写字母+小写字母+数字组合。比如：1234Qwer

![](../../assets/_images/deploy/oracle/step7.png)

创建产品清单，默认

![](../../assets/_images/deploy/oracle/step8.png)

执行先决条件检查，按照静默方式的内核参数和限制设定

```bash
# 创建swap文件 bs=2300的设置的值一般为内存的1.5倍以上 
dd if=/dev/zero of=swapfree bs=32k count=65515
mkswap swapfree # 将创建的文件用做交换分区
swapon swapfree # 开启交换分区
# 使Swap文件永久生效,/etc/fstab加入配置
echo "swapfree   swap    swap    sw  0   0" >> /etc/fstab
```

![](../../assets/_images/deploy/oracle/step9.png)

准备安装

![](../../assets/_images/deploy/oracle/step10.png)

进行安装

![](../../assets/_images/deploy/oracle/step11.png)


进度70% `ins_emagent.mk错误弹框`

![](../../assets/_images/deploy/oracle/step11_error.png)


> 编辑 /home/oracle/app/oracle/product/11.2.0/dbhome_1/sysman/lib/ins_emagent.mk 约176行，可以搜索$(MK_EMAGENT_NMECTL) 关键字快速找到。

![](../../assets/_images/deploy/oracle/step11_update.png)

```bash
#===========================
#  emdctl
#===========================

$(SYSMANBIN)emdctl:
        $(MK_EMAGENT_NMECTL) -lnnz11

#===========================
#  nmocat
#===========================
```

修改完成后，点击重试（R）

![](../../assets/_images/deploy/oracle/step11_goon.png)

数据库创建完成

![](../../assets/_images/deploy/oracle/step11_done.png)

执行配置脚本

![](../../assets/_images/deploy/oracle/step11_shell.png)

使用root身份执行

```bash
/home/oracle/app/oraInventory/orainstRoot.sh 
/home/oracle/app/oracle/product/11.2.0/dbhome_1/root.sh 
```

执行完成这两个脚本，点击确定

![](../../assets/_images/deploy/oracle/step11_shell_ok.png)

Oracle Database 安装成功

![](../../assets/_images/deploy/oracle/step12.png)

#### 1.2.6 设置oracle用户环境变量

```bash
su - oracle
vim .bash_profile

export ORACLE_HOME=/home/oracle/app/oracle/product/11.2.0/dbhome_1/
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH

source .bash_profile
```

#### 1.2.7 登录数据库

```bash 
su - oracle
lsnrctl status
lsnrctl start
lsnrctl stop
sqlplus / as sysdba
startup
shutdown immediate
select status from v$instance;
exit
```

### 1.3 DataGuard主备

Data Gurad 通过冗余数据来提供数据保护，Data Gurad 通过日志同步机制保证冗余数据和主数之前的同步，这种同步可以是实时，延时，同步，异步多种形式。Data Gurad 常用于异地容灾和小企业的高可用性方案，虽然可以在Standby 机器上执行只读查询，从而分散Primary 数据库的性能压力，但是Data Gurad 决不是性能解决方案

#### 1.3.1 搭建环境

| **名称** | **主库** | **备库** |
| ---------- | ---------- | ---------- |
| 主机名  | oracle11g | oracle11gstandby |
| 操作系统  | CentOS release 7.9 | CentOS release 7.9 |
| IP地址  | 192.168.2.201 | 192.168.2.202 |
| ORACLE_BASE  | /u01/app/oracle | /u01/app/oracle |
| ORACLE_HOME  | /u01/app/oracle/product/11.2.0/dbhome_1 | /u01/app/oracle/product/11.2.0/dbhome_1 |
| ORACLE_SID  | orcl | orcl |
| 归档模式  | 是 | 否 |
| 数据库安装  | 安装数据库软件，创建监听，建库 | 安装数据库软件，创建监听，不建库 |


#### 1.3.2 主库配置

1. 开启归档模式

[参考：oracle开启关闭归档以及归档空间满的处理方法总结](deploy/oracle?id=_24-归档日志)

2. 开启强日志模式

```bash
su - oracle
sqlplus / as sysdba
alter database force logging;
```

查看

```bash
select name,log_mode,force_logging from v$database;
```

3. 创建standby redolog日志组

```lua
1：standby redo log的文件大小与primary 数据库online redo log 文件大小相同
2：standby redo log日志文件组的个数依照下面的原则进行计算：
Standby redo log组数公式>=(每个instance日志组个数+1)*instance个数
假如只有一个节点，这个节点有三组redolog，
所以Standby redo log组数>=(3+1)*1 == 4
所以至少需要创建4组Standby redo log
```

查看当前线程与日志组的对应关系及日志组的大小：

```lua
SQL> select thread#,group#,bytes/1024/1024 from v$log;
   THREAD#     GROUP# BYTES/1024/1024
---------- ---------- ---------------
    1       1           50
    1       2           50
    1       3           50

```

如上，这里有三组redo log，所以至少需要创建4组Standby redo log，大小均为50M：

```sql
alter database add standby logfile group 4 ('/u01/app/oracle/oradata/orcl/redo_dg_04.log') size 50M;
alter database add standby logfile group 5 ('/u01/app/oracle/oradata/orcl/redo_dg_05.log') size 50M;
alter database add standby logfile group 6 ('/u01/app/oracle/oradata/orcl/redo_dg_06.log') size 50M;
alter database add standby logfile group 7 ('/u01/app/oracle/oradata/orcl/redo_dg_07.log') size 50M;
```

若删除组：

```bash
alter database drop standby logfile group x;
```

查看standy日志组的信息：

```lua
SQL> select group#,sequence#,status, bytes/1024/1024 from v$standby_log;
    GROUP#  SEQUENCE# STATUS	 BYTES/1024/1024
---------- ---------- ---------- ---------------
	 4	    0 UNASSIGNED	      50
	 5	    0 UNASSIGNED	      50
	 6	    0 UNASSIGNED	      50
	 7	    0 UNASSIGNED	      50
```

4. 创建主库密码文件

```bash
su - oracle
orapwd file=$ORACLE_HOME/dbs/orapw$ORACLE_SID password=oracle force=y
```

5. 配置spfile文件

```lua
SQL> show parameter spfile;
NAME    TYPE    VALUE
------------------------------------ ----------- ------------------------------
spfile string   /u01/app/oracle/product/11.2.0/dbhome_1/dbs/spfileorcl.ora
```

读入数据代码示例

```bash
data = pd.read_csv(
    'https://labfile.oss.aliyuncs.com/courses/1283/adult.data.csv')
print(data.head())
```

用spfile创建一个pfile,用于修改：

```bash
sqlplus / as sysdba
create pfile='/tmp/initorcl.ora' from spfile;
```

修改pfile文件

```bash
vim /tmp/initorcl.ora
```

```conf
orcl.__db_cache_size=654311424
orcl.__java_pool_size=16777216
orcl.__large_pool_size=16777216
orcl.__oracle_base='/u01/app/oracle'#ORACLE_BASE set from environment
orcl.__pga_aggregate_target=637534208
orcl.__sga_target=956301312
orcl.__shared_io_pool_size=0
orcl.__shared_pool_size=251658240
orcl.__streams_pool_size=0
*.audit_file_dest='/u01/app/oracle/admin/orcl/adump'
*.audit_trail='NONE'
*.compatible='11.2.0.0.0'
*.control_files='/u01/app/oracle/oradata/orcl/control01.ctl','/u01/app/oracle/flash_recovery_area/orcl/control02.ctl'
*.db_block_size=8192
*.db_domain=''
*.db_name='orcl'
*.db_recovery_file_dest='/u01/app/oracle/flash_recovery_area'
*.db_recovery_file_dest_size=4070572032
*.diagnostic_dest='/u01/app/oracle'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=orclXDB)'
*.log_archive_format='archive_%t_%s_%r.log'
*.memory_target=1588592640
*.open_cursors=300
*.processes=150
*.remote_login_passwordfile='EXCLUSIVE'
*.undo_tablespace='UNDOTBS1'
*.db_unique_name='orclpr'
*.fal_client='orclpr'
*.fal_server='orcldg'
*.standby_file_management='AUTO'
*.log_archive_config='DG_CONFIG=(orclpr,orcldg)'
*.log_archive_dest_1='location=/u01/app/oracle/oradata/orcl/archivelog'
*.log_archive_dest_2='SERVICE=orcldg LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=orcldg'
*.log_archive_dest_state_1='ENABLE'
*.log_archive_dest_state_2='ENABLE'
```

复制pfile文件到spfile:
```bash
shutdown immediate;
create spfile from pfile='/tmp/initorcl.ora';
startup;
```

6. 修改监听文件，添加静态监听

```bash
vi $ORACLE_HOME/network/admin/listener.ora
```

```conf
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
      (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.2.201)(PORT = 1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = orcl)
      (ORACLE_HOME = /u01/app/oracle/product/11.2.0/dbhome_1)
      (SID_NAME = orcl)
    )
  )


ADR_BASE_LISTENER = /u01/app/oracle
SAVE_CONFIG_ON_STOP_LISTENER = ON
```

重启监听服务

```bash
lsnrctl stop
lsnrctl start  
#或
lsnrctl reload
```

7. 编辑网络服务名配置

```bash
vi $ORACLE_HOME/network/admin/tnsnames.ora
```

添加配置

```conf
orclpr =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.2.201)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl)
    )
  )
orcldg =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.2.202)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl)
    )
  )
```

tnsping测试：

```lua
[oracle@oracledb dbs]$ lsnrctl reload

LSNRCTL for Linux: Version 11.2.0.1.0 - Production on 16-JAN-2024 23:12:50

Copyright (c) 1991, 2009, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1521)))
The command completed successfully
[oracle@oracledb dbs]$ vi $ORACLE_HOME/network/admin/tnsnames.ora
[oracle@oracledb dbs]$ tnsping orclpr

TNS Ping Utility for Linux: Version 11.2.0.1.0 - Production on 16-JAN-2024 23:15:57

Copyright (c) 1997, 2009, Oracle.  All rights reserved.

Used parameter files:
/u01/app/oracle/product/11.2.0/dbhome_1/network/admin/sqlnet.ora


Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = orcl)))
OK (0 msec)
[oracle@oracledb dbs]$
```

#### 1.3.3 备库配置

1. 将主库中的密码文件、pfile文件、监听文件复制到备库中：

进入主库`192.168.2.201`

```bash
su - oracle
cd /u01/app/oracle/product/11.2.0/dbhome_1/dbs
scp orapworcl 192.168.2.202:/u01/app/oracle/product/11.2.0/dbhome_1/dbs/
scp /tmp/initorcl.ora 192.168.2.202:/tmp/
cd /u01/app/oracle/product/11.2.0/dbhome_1/network/admin
scp listener.ora 192.168.2.202:/u01/app/oracle/product/11.2.0/dbhome_1/network/admin/
scp tnsnames.ora 192.168.2.202:/u01/app/oracle/product/11.2.0/dbhome_1/network/admin/
```

2. 配置spfile文件

进入从库`192.168.2.202`

```bash
su - oracle
vim /tmp/initorcl.ora
```

```conf
orcl.__db_cache_size=654311424
orcl.__java_pool_size=16777216
orcl.__large_pool_size=16777216
orcl.__oracle_base='/u01/app/oracle'#ORACLE_BASE set from environment
orcl.__pga_aggregate_target=637534208
orcl.__sga_target=956301312
orcl.__shared_io_pool_size=0
orcl.__shared_pool_size=251658240
orcl.__streams_pool_size=0
*.audit_file_dest='/u01/app/oracle/admin/orcl/adump'
*.audit_trail='NONE'
*.compatible='11.2.0.0.0'
*.control_files='/u01/app/oracle/oradata/orcl/control01.ctl','/u01/app/oracle/flash_recovery_area/orcl/control02.ctl'
*.db_block_size=8192
*.db_domain=''
*.db_name='orcl'
*.db_recovery_file_dest='/u01/app/oracle/flash_recovery_area'
*.db_recovery_file_dest_size=4070572032
*.diagnostic_dest='/u01/app/oracle'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=orclXDB)'
*.log_archive_format='archive_%t_%s_%r.log'
*.memory_target=1588592640
*.open_cursors=300
*.processes=150
*.remote_login_passwordfile='EXCLUSIVE'
*.undo_tablespace='UNDOTBS1'
*.db_unique_name='orcldg'
*.fal_client='orcldg'
*.fal_server='orclpr'
*.standby_file_management='AUTO'
*.log_archive_config='DG_CONFIG=(orclpr,orcldg)'
*.log_archive_dest_1='location=/u01/app/oracle/oradata/orcl/archivelog'
*.log_archive_dest_2='SERVICE=orclpr LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=orclpr'
*.log_archive_dest_state_1='ENABLE'
*.log_archive_dest_state_2='ENABLE'
```

复制pfile文件到spfile：

```bash
sqlplus / as sysdba
shutdown immediate;
create spfile from pfile='/tmp/initorcl.ora';
startup nomount;
```

3. 修改监听文件

```bash
vi $ORACLE_HOME/network/admin/listener.ora
```

```conf
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
      (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.2.202)(PORT = 1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = orcl)
      (ORACLE_HOME = /u01/app/oracle/product/11.2.0/dbhome_1)
      (SID_NAME = orcl)
    )
  )


ADR_BASE_LISTENER = /u01/app/oracle
```

重启监听服务

```bash
lsnrctl stop
lsnrctl start  
#或
lsnrctl reload
```

4. tnsping测试

```lua
[oracle@oracledb ~]$ tnsping orclpr

TNS Ping Utility for Linux: Version 11.2.0.1.0 - Production on 16-JAN-2024 23:44:32

Copyright (c) 1997, 2009, Oracle.  All rights reserved.

Used parameter files:
/u01/app/oracle/product/11.2.0/dbhome_1/network/admin/sqlnet.ora


Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.2.201)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = orcl)))
OK (0 msec)
[oracle@oracledb ~]$ tnsping orcldg

TNS Ping Utility for Linux: Version 11.2.0.1.0 - Production on 16-JAN-2024 23:44:33

Copyright (c) 1997, 2009, Oracle.  All rights reserved.

Used parameter files:
/u01/app/oracle/product/11.2.0/dbhome_1/network/admin/sqlnet.ora


Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.2.202)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = orcl)))
OK (0 msec)
[oracle@oracledb ~]$ 
```

5. 创建所需的目录

```bash
su - oracle
mkdir -p /u01/app/oracle/admin/orcl/adump
mkdir -p /u01/app/oracle/admin/orcl/dbdump
mkdir -p /u01/app/oracle/admin/orcl/pfile
mkdir -p /u01/app/oracle/oradata/orcl
mkdir -p /u01/app/oracle/fast_recovery_area/orcl
mkdir -p /u01/app/oracle/oradata/orcl/archivelog
```

6. 启动备库到nomount

```bash
shutdown immediate;
startup nomount;
```

7. 利用RMAN在备库上恢复主库

```
rman target sys/password@orclpr auxiliary sys/password@orcldg

duplicate target database for standby from active database nofilenamecheck;
```


> 此处必须是主库打开，备库挂载。如果主库是挂载状态，使用下面命令打开

```bash
alter database mount;
alter database open;
```

> 如果安装后忘记管理员密码，主备库分别使用以下命令重置管理员密码

```bash
ALTER USER SYS IDENTIFIED BY "password";
ALTER USER SYSTEM IDENTIFIED BY "password";
```

还原过程如下：

```lua
[oracle@oracledb ~]$ rman target sys/password@orclpr auxiliary sys/password@orcldg

Recovery Manager: Release 11.2.0.1.0 - Production on Wed Jan 17 00:11:50 2024

Copyright (c) 1982, 2009, Oracle and/or its affiliates.  All rights reserved.

connected to target database: ORCL (DBID=1685790589)
connected to auxiliary database: ORCL (not mounted)

RMAN> duplicate target database for standby from active database nofilenamecheck;

Starting Duplicate Db at 17-JAN-24
using target database control file instead of recovery catalog
allocated channel: ORA_AUX_DISK_1
channel ORA_AUX_DISK_1: SID=19 device type=DISK

contents of Memory Script:
{
   backup as copy reuse
   targetfile  '/u01/app/oracle/product/11.2.0/dbhome_1/dbs/orapworcl' auxiliary format 
 '/u01/app/oracle/product/11.2.0/dbhome_1/dbs/orapworcl'   ;
}
executing Memory Script

Starting backup at 17-JAN-24
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=35 device type=DISK
Finished backup at 17-JAN-24

contents of Memory Script:
{
   backup as copy current controlfile for standby auxiliary format  '/u01/app/oracle/oradata/orcl/control01.ctl';
   restore clone controlfile to  '/u01/app/oracle/flash_recovery_area/orcl/control02.ctl' from 
 '/u01/app/oracle/oradata/orcl/control01.ctl';
}
executing Memory Script

Starting backup at 17-JAN-24
using channel ORA_DISK_1
channel ORA_DISK_1: starting datafile copy
copying standby control file
output file name=/u01/app/oracle/product/11.2.0/dbhome_1/dbs/snapcf_orcl.f tag=TAG20240117T001310 RECID=1 STAMP=1158451990
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:01
Finished backup at 17-JAN-24

Starting restore at 17-JAN-24
using channel ORA_AUX_DISK_1

channel ORA_AUX_DISK_1: copied control file copy
Finished restore at 17-JAN-24

contents of Memory Script:
{
   sql clone 'alter database mount standby database';
}
executing Memory Script

sql statement: alter database mount standby database

contents of Memory Script:
{
   set newname for tempfile  1 to 
 "/u01/app/oracle/oradata/orcl/temp01.dbf";
   switch clone tempfile all;
   set newname for datafile  1 to 
 "/u01/app/oracle/oradata/orcl/system01.dbf";
   set newname for datafile  2 to 
 "/u01/app/oracle/oradata/orcl/sysaux01.dbf";
   set newname for datafile  3 to 
 "/u01/app/oracle/oradata/orcl/undotbs01.dbf";
   set newname for datafile  4 to 
 "/u01/app/oracle/oradata/orcl/users01.dbf";
   backup as copy reuse
   datafile  1 auxiliary format 
 "/u01/app/oracle/oradata/orcl/system01.dbf"   datafile 
 2 auxiliary format 
 "/u01/app/oracle/oradata/orcl/sysaux01.dbf"   datafile 
 3 auxiliary format 
 "/u01/app/oracle/oradata/orcl/undotbs01.dbf"   datafile 
 4 auxiliary format 
 "/u01/app/oracle/oradata/orcl/users01.dbf"   ;
   sql 'alter system archive log current';
}
executing Memory Script

executing command: SET NEWNAME

renamed tempfile 1 to /u01/app/oracle/oradata/orcl/temp01.dbf in control file

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

Starting backup at 17-JAN-24
using channel ORA_DISK_1
channel ORA_DISK_1: starting datafile copy
input datafile file number=00001 name=/u01/app/oracle/oradata/orcl/system01.dbf
output file name=/u01/app/oracle/oradata/orcl/system01.dbf tag=TAG20240117T001317
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:15
channel ORA_DISK_1: starting datafile copy
input datafile file number=00002 name=/u01/app/oracle/oradata/orcl/sysaux01.dbf
output file name=/u01/app/oracle/oradata/orcl/sysaux01.dbf tag=TAG20240117T001317
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:07
channel ORA_DISK_1: starting datafile copy
input datafile file number=00003 name=/u01/app/oracle/oradata/orcl/undotbs01.dbf
output file name=/u01/app/oracle/oradata/orcl/undotbs01.dbf tag=TAG20240117T001317
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:01
channel ORA_DISK_1: starting datafile copy
input datafile file number=00004 name=/u01/app/oracle/oradata/orcl/users01.dbf
output file name=/u01/app/oracle/oradata/orcl/users01.dbf tag=TAG20240117T001317
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:01
Finished backup at 17-JAN-24

sql statement: alter system archive log current

contents of Memory Script:
{
   switch clone datafile all;
}
executing Memory Script

datafile 1 switched to datafile copy
input datafile copy RECID=1 STAMP=1158452021 file name=/u01/app/oracle/oradata/orcl/system01.dbf
datafile 2 switched to datafile copy
input datafile copy RECID=2 STAMP=1158452021 file name=/u01/app/oracle/oradata/orcl/sysaux01.dbf
datafile 3 switched to datafile copy
input datafile copy RECID=3 STAMP=1158452021 file name=/u01/app/oracle/oradata/orcl/undotbs01.dbf
datafile 4 switched to datafile copy
input datafile copy RECID=4 STAMP=1158452021 file name=/u01/app/oracle/oradata/orcl/users01.dbf
Finished Duplicate Db at 17-JAN-24
```

8. 登陆备库并查看数据库当前状态

```bash
sqlplus / as sysdba

SQL> select status from v$instance;

STATUS
------------
MOUNTED
```

RMAN恢复完直接就是mount状态

9. 备库启动日志应用

```bash
alter database recover managed standby database disconnect from session;

SQL> select sequence#,applied from v$archived_log order by 1;

 SEQUENCE# APPLIED
---------- ---------
    11 YES
    12 YES
    13 YES
```

10. 分别查看主库和备库的归档序列号是否一致

先在主库手动切换一下日志再查看：

```bash
SQL> alter system switch logfile;

System altered.

SQL> archive log list;
Database log mode	       Archive Mode
Automatic archival	       Enabled
Archive destination	       /u01/app/oracle/oradata/orcl/archivelog
Oldest online log sequence     13
Next log sequence to archive   15
Current log sequence	       15
SQL> 
```

再在备库上查看：

```bash
SQL> archive log list;
Database log mode	       Archive Mode
Automatic archival	       Enabled
Archive destination	       /u01/app/oracle/oradata/orcl/archivelog
Oldest online log sequence     13
Next log sequence to archive   0
Current log sequence	       15
```

11. 查看备库中各文件如下

```bash
cd /u01/app/oracle/oradata/orcl
```

至此，dataguard已部署完成，可以测试是否成功！

> 如果从库出现无法登录的情况，提示`ORA-10456:cannot open standby database;media recovery session may be in progress`

```bash
alter database recover managed standby database cancel;     # 先在从库停止standby
alter database open;
alter database recover managed standby database using current logfile disconnect;   # 再启动日志应用
```

#### 1.3.4 主备切换


### 1.4 卸载

```bash
su - oracle 
sqlplus / as sysdba
shutdown immediate
exit
lsnrctl stop
cd /u01/app/oracle/product/11.2.0/dbhome_1/deinstall
./deinstall
rm -rf /u01/app
```

## 2. 库操作

### 2.1 用户管理

1. 创建用户

```sql
create user xzh0610 identified by 123456
  default tablespace xzh
  temporary tablespace xzh_temp
  profile DEFAULT
  password  expire;

grant connect,resource to xzh0610;
grant read,write ON DIRECTORY oradmp to xzh0610; 
grant dba to xzh0610;
```

2. 删除用户

```sql
drop user xzh0610 cascade;
```

3. 查询用户

```sql
select * from sys.all_users order by user_id;
```

4. 修改密码

```bash
# 直接修改
password zhangsan
# 使用SQL语句查找密码过期用户所属的profile
select username,profile from dba_users;  
# 查看对应的概要文件(如default)的密码有效期设置（一般默认为180天）
SELECT * FROM dba_profiles s WHERE s.profile='DEFAULT' AND resource_name='PASSWORD_LIFE_TIME';
# 然后使用SQL语句将该用户所属的profile修改为永不过期
alter profile default limit PASSWORD_LIFE_TIME unlimited;
# 将密码过期用户的密码更新，使用如下SQL
alter user zhangsan identified by "密码" account unlock;
# 解锁用户
alter user zhangsan account unlock;
```

### 2.2 表空间管理

1. 临时表空间
  
表空间名字不能重复，即便存储的位置不一致, 但是dbf文件可以一致，50m为表空间的大小，对大数据量建议32G

```sql
create temporary tablespace xzh_temp
tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' 
size 50m
autoextend on 
next 50m maxsize 20480m 
extent management local;
```

```sql
select tablespace_name,file_name,bytes/1024/1024 file_size,autoextensible from dba_temp_files;          -- 查询临时表空间
create temporary tablespace xzh_temp tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' size 10M;          -- 创建临时表空间
alter database tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' resize 100M;                             -- 调整临时表空间大小
alter tablespace xzh_temp add tempfile '/u01/app/oracle/oradata/xzh_temp_2.dbf' size 100m;                  -- 向临时表空间中添加数据文件： 
alter database tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' autoextend on next 5m maxsize unlimited;     -- 将临时数据文件设为自动扩展
alter database tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' drop;                                        -- 删除临时表空间的一个数据文件
drop tablespace xzh_temp including contents and datafiles cascade constraints;                              -- 删除临时表空间(彻底删除)
```

2. 数据表空间

```sql
create tablespace xzh
logging 
datafile '/u01/app/oracle/oradata/xzh.dbf' 
size 50m 
autoextend on 
next 50m maxsize 20480m 
extent management local;
```

```sql
select tablespace_name, sum(bytes)/1024/1024 from dba_data_files group by tablespace_name;            --  查询表空间
create tablespace xzh datafile '/u01/app/oracle/xzh/xzh.dbf' size 100M;                               --  创建表空间
alter tablespace xzh add datafile '/u01/app/oracle/xzh/xzh.dbf' size 5m;                              --  增加表空间数据文件
drop tablespace xzh including contents;                                                           -- 删除表空间
drop tablespace xzh including contents and datafiles;                                             -- 删除表空间和数据文件
alter database datafile '/u01/app/oracle/xzh/xzh.dbf' resize 50m;                                 -- 扩展表空间
alter database datafile '/u01/app/oracle/xzh/xzh.dbf' autoextend on next 50m maxsize 500m;        -- 表空间自动增长
```

```sql
-- 查询表空间利用率
select b.file_name 物理文件名,
       b.tablespace_name 表空间,
       b.bytes / 1024 / 1024 大小M,
       (b.bytes - sum(nvl(a.bytes, 0))) / 1024 / 1024 已使用M,
       substr((b.bytes - sum(nvl(a.bytes, 0))) / (b.bytes) * 100, 1, 5) 利用率
  from dba_free_space a, dba_data_files b
 where a.file_id = b.file_id
 group by b.tablespace_name, b.file_name, b.bytes
 order by b.tablespace_name;
-- 查询用户隶属表空间
select username,default_tablespace from dba_users where username='xzh';
-- 增加表空间文件
alter tablespace xzh_data ADD datafile '/u01/app/oracle/xzh/xzh.dbf' size 1024M autoextend on next 1024M maxsize 32767M; 
```


### 2.3 审计日志

#### 2.3.1 关闭审计

```bash
sqlplus /nolog
connect / as sysdba
show parameter audit_trail;                         # VALUE值为DB，表示审计功能为开启的状态
alter system set audit_trail=none scope=spfile;     # 关闭审计功能
shutdown immediate;                                 # 重启数据库
startup;
show parameter audit_trail;                         # VALUE值为NONE，表示审计功能已关闭
truncate table SYS.AUD$;                            # 删除审计日志
```

#### 2.3.2 开启审计

11g默认是开始审计的，有审计记录，所以不需要安装,如果查询发现表不存在,则需要安装。

```bash
SQL> @/u01/app/oracle/product/11.2.0/dbhome_1/rdbms/admin/cataudit.sql;
```

```bash
alter system set audit_sys_operations=true scope=spfile;                    # 设置审计系统级操作
alter system set audit_trail=db,extended scope=spfile;                      # 设置审计日志记录到数据库表中，并包含扩展信息
alter system set audit_file_dest='/u01/app/oracle/audit_logs' cope=spfile;  # 设置审计日志文件的存储路径
shutdown immediate;         # 重启数据库
startup;
show parameter audit;       # 查看数据库审计配置信息
select * FROM SYS.AUD$;
```

#### 2.3.3 审计迁移

审计表默认安装在SYSTEM表空间,在生产环境一般都建议迁移到其他表空间里面，步骤如下

```bash
create tablespace shenji logging datafile '/u01/app/oracle/oradata/ORDB/shenji.dbf' size 200m autoextend off extent management local segment space management auto;

alter table aud$ move tablespace shenji;
alter table audit$ move tablespace shenji;
alter index i_audit rebuild online tablespace shenji;
alter table audit_actions move tablespace shenji;
alter index i_audit_actions rebuild online tablespace shenji;

select bytes/1024/1024 MB from dba_segments where segment_name='AUD$';          # 查看审计日志大小
select table_name,tablespace_name from dba_tables where table_name like '%AUD%';
select index_name,tablespace_name from dba_indexes where index_name like '%AUDIT%';
```


### 2.4 归档日志

#### 2.4.1 查看归档模式

```bash
sqlplus / as sysdba
archive log list
```

下文显示未开启

```lua
Database log mode No Archive Mode
Automatic archival Disabled
Archive destination USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence 8
Current log sequence 10
```

#### 2.4.2 开启归档模式

```bash
shutdown immediate;             # 关闭实例
startup mount;                  # 启动到mount
alter database archivelog;      # 开启归档模式
archive log list; 
```

再次查看是否开启归档，下文显示已归档

```lua
Database log mode Archive Mode
Automatic archival Enabled
Archive destination USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence 8
Next log sequence to archive 10
Current log sequence 10
```

```bash
alter database open;            # 打开数据库
show parameter db_recovery;     # 查看参数db_recovery_file_dest归档日志目录(默认闪回恢复区)、db_recovery_file_dest_size大小
```

默认情况下，归档日志会存放到USE_DB_RECOVERY_FILE_DEST(闪回恢复区flash_recovery_area)内，如果闪回恢复区已满，归档日志就有可能无法继续归档，通常的解决方法是增大闪回恢复区，可以用以下SQL实现：

```bash
alter system set db_recovery_file_dest_size=3G;
```

从10g开始，可以设置多个归档路径，生成多份一样的日志

```bash
su - oracle 
mkdir /u01/app/oracle/oradata/orcl/archivelog
sqlplus / as sysdba
alter system set log_archive_dest_1 = 'location=/u01/app/oracle/oradata/orcl/archivelog';
show parameter log_archive_dest
archive log list;
select created, log_mode from v$database;       # 查看归档方式
```

```bash
# 查看归档日志位置
show parameter log_archive_dest;
# 查看归档日志格式
show parameter log_archive_format;
# 修改归档日志格式
alter system set log_archive_format ="archive_%t_%s_%r.log" scope=spfile;
shutdown immediate;
startup;
# 查看归档日志进程数
show parameter log_archive_max_process;
# 归档当前重做日志
alter system archive log current;       # 是归档当前的重做日志文件，不管自动归档有没有打都归档。
select name from v$archived_log;
```

#### 2.4.3 删除归档日志

先手动删除物理文件

```bash
find /u01/app/oracle/flash_recovery_area/ORCL/archivelog -xdev -mtime +7 -name "*.dbf" -exec rm -f {} ; 
# 或
find /u01/app/oracle/oradata/orcl/archivelog -type f -mtime +7 -exec rm {} ;
```

再执行以下命令

```bash
su - oracle
rman target /
list archivelog all;        # 列出日志目录
delete archivelog all completed before 'sysdate-7';     # 删除所有日志使用：sysdate
```

最后会在RMAN里留下未管理的归档文件，要在RMAN里执行下面2条命令

```bash
crosscheck archivelog all;      # 检查归档日志
delete expired archivelog all;  # 删除失效日志及
crosscheck archivelog all;      # 检查归档日志
```

如果因为日志磁盘满载不能执行命令，可以强制删除所有日志
```bash
delete noprompt force archivelog all;
```

#### 2.4.4 关闭归档模式

```bash
sqlplus / as sysdba
shutdown immediate;
startup mount;
alter database noarchivelog;
alter database open;
archive log list;
```


### 2.5 目录管理

```sql
SELECT * FROM DBA_DIRECTORIES;                          -- 查看目录
CREATE DIRECTORY oradmp AS '/u01/app/oracle/oradmp';    -- 创建目录
DROP DIRECTORY oradmp;                                  -- 删除目录
GRANT READ,WRITE ON DIRECTORY oradmp to xzh0610;        -- 将oradmp目录的赋给用户
```


### 2.6 备份恢复

1. 按表名备份、还原

```bash
expdp xzh0610/123456 directory=oradmp dumpfile=xzh0610.dmp tables=sys_menu,sys_role,sys_user  
impdp xzh0610/123456 directory=oradmp dumpfile=xzh0610.dmp tables=xzh0610.sys_menu,xzh0610.sys_user REMAP_SCHEMA=xzh0610:xzh0610 table_exists_action=replace
```

2. 全量备份、还原

```bash
expdp xzh0610/123456 directory=oradmp dumpfile=xzh0610.dmp SCHEMAS=xzh0610 logfile=xzh0610_$(date +%Y%m%d-%H%M).log
impdp xzh0611/123456 directory=oradmp dumpfile=xzh0610.dmp  schemas=xzh0610 REMAP_SCHEMA=xzh0610:xzh0611 REMAP_TABLESPACE=xzh:xzh
```

> 还原成功调用报错 `ORA-00600: internal error code, arguments: [19004]` ，解决办法

```bash
execute dbms_stats.delete_schema_stats('xzh0610');
```

3. 定时备份

```bash
crontab -e
30 3 * * * sh /data/shell/bakup.sh  # 每天凌晨3点半执行
```

```bash
#!/bin/bash
starttime=`date +'%Y-%m-%d %H:%M:%S'`
stime=`date +'%Y-%m-%d %H:%M:%S'`
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '总计开始时间：'$starttime >> /data/kh_shell/kh_log.log
BAKDATE=`date +'%Y-%m-%d'`
DB_NAME=VJSP_JSWZ_190601
echo ${BAKDATE}
echo $DB_NAME$BAKDATE
cd /data/bakup/KHDB

stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '清空临时表空间开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
alter tablespace TEMP shrink space;
alter tablespace JSWZ_TEMP shrink space;
disconnect
quit
EOF"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '清空临时表空间结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 2s
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '删库解压授权开始时间：'$stime >> /data/kh_shell/kh_log.log
rm -rf /data/bakup/KHDB/*.dmp
unzip $DB_NAME$BAKDATE".zip"
chmod -R 777 /data/bakup/KHDB/*
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '删库解压授权结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 2s;
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '还原数据库1开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "impdp VJSP_JSWZ_190601/wzld9999 directory=KH_DIR dumpfile=VJSP_JSWZ_${BAKDATE}_%U.dmp parallel=4 logfile=1_${BAKDATE}.log remap_schema=VJSP_JSWZ_190601:VJSP_JSWZ_190601 TABLE_EXISTS_ACTION=TRUNCATE tables=VJSP_JS_TWOHOURBACKINFO,TS_FLOW_CALENDAR,ZJ_CASE_INFO,VJSP_JS_GRIDSETTING,VJSP_JS_ASSMENTDEPTS,VJSP_JS_PARTSINFO,VJSP_JS_GDALINFO,VJSP_JS_DELAYEDINFO,VJSP_JS_HANGUPINFO,VJSP_JS_MARKRECORD,VJSP_JS_URGEWITHDRAWREPAIR,VJSP_JS_QUALITYGDUSERDEAIL,VJSP_HR_STAFF_BASE,VJSP_JS_DECLAREPOINT,VJSP_DEPT,VJSP_JS_goahead_visit,vjsp_js_gduserinfo"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '还原数据库1结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 5s;
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '还原数据库2开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "impdp VJSP_JSWZ_190601/wzld9999 directory=KH_DIR dumpfile=VJSP_JSWZ_${BAKDATE}_%U.dmp parallel=4 logfile=5_${BAKDATE}.log remap_schema=VJSP_JSWZ_190601:VJSP_JSWZ_190601 TABLE_EXISTS_ACTION=TRUNCATE tables=TS_SYSTEM_DYWJ"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '还原数据库2结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 5s;
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '脚本1开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
@/data/kh_shell/update_db.sql;
commit;
disconnect
quit
EOF"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '脚本1结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 5s;
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '脚本2开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
call PROC_DELETE_PATHANDDYWJ();
commit;
disconnect
quit
EOF"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '脚本2结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 5s;
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '重启数据库开始时间：'$stime >> /data/kh_shell/kh_log.log
sleep 5s
su - oracle -c "sqlplus /nolog<< EOF
conn /as sysdba
shutdown immediate
quit
EOF"

sleep 5s
su - oracle -c "sqlplus /nolog<< EOF
conn /as sysdba
startup
quit
EOF"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '重启数据库结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 5s
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '删除文件开始时间：'$stime >> /data/kh_shell/kh_log.log
find /data/bakup/KHDB/ -mtime +1 -name "*.dmp" | xargs rm -f
find /data/bakup/KHDB/ -mtime +1 -name "*.zip" | xargs rm -f
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '删除文件结束时间：'$etime >> /data/kh_shell/kh_log.log

endtime=`date +'%Y-%m-%d %H:%M:%S'`
echo '总计结束时间：'$endtime >> /data/kh_shell/kh_log.log
echo "本次总计运行时间： "$(($(date --date="$endtime" +%s)-$(date --date="$starttime" +%s)))"s" >> /data/kh_shell/kh_log.log
```


```bash
#!/bin/bash
BAKDATE=`date +%Y%m%d%H%M`
/u01/app/oracle/product/11.2.0/dbhome_1/bin/expdp SGDB/123456 SCHEMAS=SGDB DIRECTORY=DP_DEMO dumpfile=SGDB$BAKDATE.dmp logfile=SGDB$BAKDATE.log
/usr/bin/gzip /opt/DB/SGDB$BAKDATE.dmp
find /opt/DB/ -mtime +5 -type f -name '*.dmp.gz' -exec rm {} \;
find /opt/DB/ -mtime +5 -type f -name '*.log' -exec rm {} \;
```

> 如何停止正在后台执行的 impdp/expdp 任务

```sql
-- 删除所有非执行中的任务
select 'drop table ' || owner_name || '.' || job_name || ';' from dba_datapump_jobs where state = 'NOT RUNNING';
-- 查询正在执行的任务
select * from dba_datapump_jobs where STATE='EXECUTING';
```

```bash
# 根据上面 job_name 进入到刚才执行的expdp下
expdp user_center/123456 attach=SYS_EXPORT_SCHEMA_01
# 执行 stop_job=immediate ---yes 或者 kill_job
```

### 2.7 执行计划

1. 手动执行

```sql
explain plan for 
select * from dual;
---------------------
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

2. AWR报告

手工创建快照

```sql
select dbms_workload_repository.create_snapshot() from dual;
```

导出报告

```bash
su - oracle
sqlplus /nolog
conn /as sysdba
@?/rdbms/admin/awrrpt.sql

# 要求填写要生成的报告格式，支持html和text，html是默认值可直接回车
# 输入要导出最近几天的报告
# 输入启和止的Snap ID
# 输入报告名称
```

3. 相关视图

```sql
-- 查看已执行sql的执行计划
select * from v$sql where LOWER(sql_text) LIKE '% from userinf2%' order by last_active_time DESC;
select * from table(dbms_xplan.display_cursor('2vd4wsytmnhvs'));

-- 执行最糟糕的SQL
select s.snap_id,
       s.module,
       s.sql_id,
       s.disk_reads_delta,
       s.executions_delta,
       s.disk_reads_delta /
       decode(s.executions_delta, 0, 1, s.executions_delta)
  from dba_hist_sqlstat s
 where s.disk_reads_delta > 10000
 order by s.disk_reads_delta desc;
-- 快照历史表
select t.snap_id,t.dbid,instance_number
      ,to_char(t.startup_time,'yyyy-mm-dd hh24:mi:ss') AS startup_time
      ,to_char(t.begin_interval_time,'yyyy-mm-dd hh24:mi:ss') AS begin_interval_time
      ,to_char(t.end_interval_time,'yyyy-mm-dd hh24:mi:ss') AS end_interval_time
  from dba_hist_snapshot t order by snap_id desc;
-- SQL文本内容历史表
select ds.sql_text
  from dba_hist_sqltext ds
 where ds.sql_id = '08u0dzcav6n8n';
```



### 2.8 常用命令

1. 服务重启

```sql
su – oracle
sqlplus /nolog
connect / as sysdba
shutdown immediate
startup
```

2. 脚本重启

```bash
vi /data/oracle_restart.sh
```

```sh
su - oracle -c "sqlplus /nolog<< EOF
conn /as sysdba
shutdown immediate
startup
quit
EOF"
```

```bash
sh /data/oracle_restart.sh
nohup sh /data/oracle_restart.sh &     # 后台执行
```

3. 关闭监听

```bash
lsnrctl status
lsnrctl
set log_status off
save_config
show log_status
stop
start
```

```sql
SELECT version FROM product_component_version WHERE substr(product, 1, 6) = 'Oracle';         -- 查看版本
select username,count(username) from v$session where username is not null group by username;  -- 查看不同用户的连接数
select count(*) from v$session where status='ACTIVE';                                         -- 查询oracle的并发连接数
select a.*,round(a.bytes/1024/1024,2) M from v$sgastat a where a.NAME = 'free memory';        -- 查询share pool的空闲内存
select count(*) from v$session;                          -- 查看数据库当前链接数
SELECT NAME FROM v$controlfile;                          -- 查看控制文件 
select value from v$parameter where name = 'processes';  -- 最大连接
alter system set processes = value scope = spfile;       -- 修改连接数需重启
```



## 3. 表操作

### 3.1 建表

```sql
create table tb_order
(
  id             NUMBER(20),
  car_no         VARCHAR2(100),
  start_price    NUMBER(10,2),
  view_num       NUMBER(5),
  on_status      NUMBER(1) default 0,
  on_time        TIMESTAMP(6),
  register_date  DATE default sysdate,
  create_user_id NUMBER(20),
  create_time    TIMESTAMP(6) WITH TIME ZONE,
  is_deleted     NUMBER(1) default 0
);

alter table tb_order add constraint pk_tb_order_id primary key (id);

comment on column tb_order.start_price  is '起拍价格';
comment on column tb_order.view_num  is '库存数量';
comment on column tb_order.on_status  is '上架状态';
comment on column tb_order.on_time  is '上架时间';
comment on column tb_order.register_date  is '注册日期';
comment on column tb_order.create_user_id  is '创建人';
comment on column tb_order.create_time  is '创建时间';
comment on column tb_order.is_deleted  is '是否删除';
```


```sql
SELECT
    * 
FROM
    (
    SELECT
        tt.*,
        ROWNUM AS RN 
    FROM
        ( SELECT t.* FROM TABLE_NAME t WHERE 1 = 1 ORDER BY CDATE DESC ) tt 
    WHERE
        ROWNUM &lt;= #{pageNum}*#{pageSize}
    ) rs 
WHERE
    rs.RN > #{pageNum-1}*#{pageSize}
```

### 3.2 锁表

```sql
-- 执行中sql
SELECT S.USERNAME,
       OBJECT_NAME,
       MACHINE,
       S.LOGON_TIME,
       L.LOCKED_MODE,
       'ALTER SYSTEM KILL SESSION ''' || S.SID || ',' || S.SERIAL# || '''' || ';'
  FROM GV$LOCKED_OBJECT L, DBA_OBJECTS O, GV$SESSION S
 WHERE L.OBJECT_ID = O.OBJECT_ID
   AND L.SESSION_ID = S.SID; 

SELECT 'ALTER SYSTEM KILL SESSION ''' || SID || ',' || SERIAL# || '''' || ';'
  FROM V$SESSION
 WHERE USERNAME = 'XW0125'

-- ora-00031错误执行：ps -ef|grep spid，kill -9 spid
select spid, osuser, s.program from v$session s, v$process p where s.paddr = p.addr and s.sid = {sid};
```

## 4. 统计信息

### 4.1 负载指标统计

#### 4.1.1 索引

```sql
-- 索引信息
select index_name, status from user_indexes where table_name = 'TABLENAME';
select index_name, status from all_indexes where table_name='DBNAME' and table_name = 'TABLENAME';
-- 索引对应字段信息
select * from user_ind_columns where table_name = 'TABLENAME';
select * from all_ind_columns where table_name='DBNAME' and table_name = 'TABLENAME';
```

#### 4.1.2 AWR

```sql
-- 清空共享内存
alter system flush shared_pool
-- 查询占用share pool 内存大于10m的sql
select substr(sql_text,1,100) "stmt",count(*) ,sum(sharable_mem),sum(users_opening),sum(executions) from v$sql group by substr(sql_text,1,100) having sum(sharable_mem)>10000000;
-- share pool的空闲内存
select a.* ,round(a.bytes/1024/1024,2) M from v$sgastat a where a.name='free memory';
-- version count过高的语句
select address,sql_id,hash_value,version_count,users_opening,users_executing,sql_text from v$sqlarea where version_count>10;
-- 获取PGA，SGA使用情况
select name,total,round(total-free,2) used,round(free,2) free,round((total-free)/total*100,2) pctused from (select 'SGA' name,(select sum(value/1024/1024) from v$sga) total,(select sum(bytes/1024/1024) from v$sgastat where name='free memory')free from dual) union select name,total,round(used,2) used,round(total-used,2) free,round(used/total*100,2)pctused from(select 'PGA' name,(select value/1024/1024 total from v$pgastat where name='aggregate PGA target parameter')total,(select value/1024/1024 used from v$pgastat where name='total PGA allocated')used from dual);
-- 获取shared pool 使用情况
select name ,round(total,2) total,round((total-free),2) used,round(free,2) free,round((total-free)/total*100,2) pctused from(select 'Shared pool' name,(select sum(bytes/1024/1024) from v$sgastat where pool='shared pool') total, (select bytes/1024/1024 from v$sgastat where name='free memory' and pool='shared pool')free from dual)

-- 查询使用频率最高的5个sql
select sql_text,executions from (select sql_text,executions,rank() over (order by executions desc) exec_rank from v$sql) where exec_rank <=5;
-- 查看cpu使用率最高的sql
select * from  (select sql_text,sql_id,cpu_time from v$sql order by cpu_time desc) where rownum<=10 order by rownum asc;
-- 消耗磁盘最多的5个sql
select disk_reads,sql_text from (select sql_text,disk_reads,dense_rank() over (order by disk_reads desc) disk_reads_rank from v$sql) where disk_reads_rank<=5;
-- 找出需要大量缓冲读取操作的查询
select buffer_gets,sql_text from (select sql_text,buffer_gets,dense_rank() over (order by buffer_gets desc) buffer_gets_rank from v$sql) where buffer_gets_rank<=5;
-- 查询数据字典缓存的命中率和缺失率
select round(((1-sum(getmisses)/(sum(gets)+sum(getmisses))))*100,3) "HR" ,round(sum(getmisses)/sum(gets)*100,3) "MR" from v$rowcache where gets+getmisses>0;
-- 查询执行最慢的sql
select *
 from (select sa.SQL_TEXT,
        sa.SQL_FULLTEXT,
        sa.EXECUTIONS "执行次数",
        round(sa.ELAPSED_TIME / 1000000, 2) "总执行时间",
        round(sa.ELAPSED_TIME / 1000000 / sa.EXECUTIONS, 2) "平均执行时间",
        sa.COMMAND_TYPE,
        sa.PARSING_USER_ID "用户ID",
        u.username "用户名",
        sa.HASH_VALUE
     from v$sqlarea sa
     left join all_users u
      on sa.PARSING_USER_ID = u.user_id
     where sa.EXECUTIONS > 0
     order by (sa.ELAPSED_TIME / sa.EXECUTIONS) desc)
 where rownum <= 50;
```

#### 4.1.3 等待事件

```sql
-- 查看当前等待事件及数量，如果是库问题 优化参数或调整业务逻辑等，如果是sql问题 继续
select inst_id,event,count(*) from gv$session_wait
where wait_class not like 'Idle'
group by inst_id, event order by 3 desc;

--查询当前执行sql
SELECT b.inst_id,
       b.sid oracleID,
       b.username,
       b.serial#,
       spid,
       paddr,
       sql_text,
       sql_fulltext,
       b.machine,
       b.EVENT,
       'alter system kill session ''' || b.sid || ',' || b.serial# || ''';'
  FROM gv$process a, gv$session b, gv$sql c
 WHERE a.addr = b.paddr
   AND b.sql_hash_value = c.hash_value
   and a.inst_id = 1
   and b.inst_id = 1
   and c.inst_id = 1
   and b.status = 'ACTIVE';

--查询当前执行sql按event分类数量
SELECT event, count(1)
  FROM gv$process a, gv$session b, gv$sql c
 WHERE a.addr = b.paddr
   AND b.sql_hash_value = c.hash_value
      --and event like '%log file switch%'
   and a.inst_id = 1
   and b.inst_id = 1
   and c.inst_id = 1
 group by event

--带入等待事件，查到当前等待事件最多的sql
SELECT b.inst_id,
       b.sid oracleID,
       b.username,
       b.serial#,
       spid,
       paddr,
       sql_text,
       sql_fulltext,
       b.machine,
       b.EVENT,
       c.SQL_ID,
       c.CHILD_NUMBER
  FROM gv$process a, gv$session b, gv$sql c
 WHERE a.addr = b.paddr
   AND b.sql_hash_value = c.hash_value
   and event like '%SQL*Net message from client%'
   and a.inst_id = 1
   and b.inst_id = 1
   and c.inst_id = 1

--iotop
select s.sql_text from v$sql s, v$session t,v$process v where s.sql_id = t.SQL_ID AND t.PADDR = v.ADDR AND v.SPID = '44196';
select S.SADDR, S.SID, S.SERIAL#, S.MACHINE, S.LOGON_TIME  from V$SESSION S where PADDR IN (select ADDR from V$PROCESS where SPID IN (55751,15842));
```

### 4.2 数据分布统计

#### 4.2.1 空间

1. 表空间使用情况

```sql
SELECT A.TABLESPACE_NAME "表空间名",
       TOTAL / (1024 * 1024) "表空间大小(M)",
       FREE / (1024 * 1024) "表空间剩余大小(M)",
       (TOTAL - FREE) / (1024 * 1024) "表空间使用大小(M)",
       ROUND((TOTAL - FREE) / TOTAL, 4) * 100 "使用率 %"
  FROM (SELECT TABLESPACE_NAME, SUM(BYTES) FREE
          FROM DBA_FREE_SPACE
         GROUP BY TABLESPACE_NAME) A,
       (SELECT TABLESPACE_NAME, SUM(BYTES) TOTAL
          FROM DBA_DATA_FILES
         GROUP BY TABLESPACE_NAME) B
 WHERE A.TABLESPACE_NAME = B.TABLESPACE_NAME;
```

2. 表占用空间

```sql
SELECT SEGMENT_NAME TABLE_NAME,
       SUM(BLOCKS) BLOCKS,
       SUM(BYTES) / (1024 * 1024) "TABLE_SIZE[MB]"
  FROM USER_SEGMENTS
 WHERE SEGMENT_TYPE = 'TABLE'
   AND SEGMENT_NAME = 'tb_name'
 GROUP BY SEGMENT_NAME;
```

3. 所有表占用空间

```sql
SELECT OWNER AS "用户名", SUM(BYTES) / 1024 / 1024 AS "所有表的大小(MB)"
FROM DBA_SEGMENTS
WHERE SEGMENT_NAME IN (SELECT T2.OBJECT_NAME
                        FROM DBA_OBJECTS T2
                       WHERE T2.OBJECT_TYPE = 'TABLE')
GROUP BY OWNER ORDER BY 2 DESC;
```

#### 4.2.2 数据

1. 每个表记录行数查询

```sql
SELECT T.TABLE_NAME "表名",T.NUM_ROWS "记录条数" FROM USER_TABLES T;
```

2. 所有表记录行数查询

```sql
SELECT SUM(NUM_ROWS) "记录总条数" FROM SYS.ALL_TABLES T WHERE T.OWNER = 'database_name';
```

3. 查询所有存储过程

```sql
SELECT * FROM ALL_OBJECTS WHERE OBJECT_TYPE = 'PROCEDURE' AND OWNER='database_name'
```

4. 按关键字查找出现的位置

```sql
SELECT * FROM USER_SOURCE WHERE UPPER(TEXT) LIKE UPPER('%keywords%');
```

## 5. PL/SQL

### 5.1 匿名块

1. 遍历

```sql
DECLARE
  -- LOCAL VARIABLES HERE
  I           INTEGER;
  BMNAMECOUNT INT := 0;
  CSNAMECOUNT INT := 0;
  V_SQL1      VARCHAR2(4000);
  V_SQL2      VARCHAR2(4000);
  R           NUMBER := -1;
BEGIN
  FOR ITEM IN (SELECT MENUID FROM VJSP_ADMIN_MENU_INFO) LOOP
    UPDATE VJSP_ADMIN_MENU_INFO
       SET MENU_BH =
           (SELECT GET_DOCUMENT_CODE('10001002') FROM DUAL)
     WHERE MENUID = ITEM.MENUID;
  END LOOP;
END;
```

2. 遍历执行SQL

```sql
DECLARE
  -- LOCAL VARIABLES HERE
  I           INTEGER;
  BMNAMECOUNT INT := 0;
  CSNAMECOUNT INT := 0;
  V_SQL1      VARCHAR2(4000);
  V_SQL2      VARCHAR2(4000);
  R           NUMBER := -1;
BEGIN
  -- TEST STATEMENTS HERE
  FOR ITEM IN (SELECT TB_TABLE_NAME
                 FROM TB_SYSTEM_ZD_MK
                WHERE TB_MK_ID IN
                      (SELECT TS_SYSTEM_MK_ID FROM TS_SYSTEM_MK_GL)
                  AND TB_LB_ID = 0
                  AND TB_T_LB_ID = 7) LOOP
    SELECT COUNT(1)
      INTO BMNAMECOUNT
      FROM USER_TAB_COLUMNS
     WHERE TABLE_NAME = ITEM.TB_TABLE_NAME
       AND COLUMN_NAME = 'TS_SPBMNAME';
    IF BMNAMECOUNT = 0 THEN
      V_SQL1 := 'ALTER TABLE ' || ITEM.TB_TABLE_NAME || ' ADD TS_SPBMNAME VARCHAR2(4000)';
      EXECUTE IMMEDIATE V_SQL1;
    END IF;
  END LOOP;
END;
```

### 5.2 函数

```sql
CREATE OR REPLACE FUNCTION FUN_OTO_ORDERBYSHOP(P_USERID IN CHAR)
  RETURN VARCHAR2 AS
  /*ECMS-订单管理查询订单按照当前登录人员的店铺查询 add by xcg 20131230*/
  V_RTN    VARCHAR2(32765) := '';
  V_SHOPID VARCHAR2(32765) := '';
  V_INDEX  INT := 0;
BEGIN
  SELECT MAX(BM.TS_SHOPID)
    INTO V_SHOPID
    FROM BM
   WHERE BM.BID = (SELECT U.USERDEPT FROM USERINF U WHERE U.ID = P_USERID); --查询当前用户所归属的店铺
  IF V_SHOPID IS NULL THEN
    --说明是OTO管理员登录 需要看见全部的数据    
    FOR ITEM IN (SELECT DISTINCT (D.TS_GS_CODE) FROM DEAL_ORDER D) LOOP
      DBMS_OUTPUT.PUT_LINE(ITEM.TS_GS_CODE);
      IF V_INDEX = 0 THEN
        V_RTN := ITEM.TS_GS_CODE;
      ELSE
        V_RTN := V_RTN || ',' || ITEM.TS_GS_CODE;
      END IF;
      V_INDEX := V_INDEX + 1;
    END LOOP;
  ELSE
    --只看到自己绑定部门下的店铺的订单信息
    V_RTN := V_SHOPID;
  END IF;
  RETURN V_RTN;
END;
```

### 5.3 存储过程

1. 执行SQL

```sql
CREATE OR REPLACE PROCEDURE PROC_updateSortCommon(V_GNID   NUMBER, --审批状态
                                                  V_MKID   NUMBER,
                                                  V_LBID   NUMBER,
                                                  V_DJID   CHAR,
                                                  V_USERID CHAR) AS
  v_TAB_SQL       VARCHAR2(4000);
  v_COL_SQL       VARCHAR2(4000);
  V_DLBID         NUMBER;
  V_DJZT          NUMBER;
  V_TABLENAME     VARCHAR2(100);
  V_TABLENAMEID   VARCHAR2(100);
  V_TABLENAMELB   VARCHAR2(100);
  V_TABLENAMEQCLB VARCHAR2(100);
  V_SQL           VARCHAR2(800);
BEGIN
  IF V_GNID = -1 THEN
    SELECT NVL(MAX(M.TS_ACTION_LB_ID), -1)
      INTO V_DLBID
      FROM TS_LB_GROUP_INFO I
     INNER JOIN TS_LB_GROUP_MX M
        ON I.TS_LB_GROUP_ID = M.TS_LB_GROUP_ID
     WHERE I.TB_SYSTEM_MK_ID = V_MKID
       AND M.TS_DEFAULT_LB_ID = V_LBID
     ORDER BY M.TS_LB_GROUP_ORDER;
    ----
    IF V_DLBID != -1 THEN
      v_TAB_SQL := 'SELECT TB_TABLE_NAME FROM TB_SYSTEM_ZD_MK  WHERE TB_MK_ID=:1 AND TB_LB_ID=:2 AND TB_T_LB_ID=:3';
      v_COL_SQL := 'SELECT TB_COLUMN_NAME FROM TB_SYSTEM_ZD_COLUMN WHERE TB_MK_ID=:1 AND TB_LB_ID=:2 AND TB_T_LB_ID=:3 AND TB_COLUMN_ID=:4';
      EXECUTE IMMEDIATE v_TAB_SQL
        INTO V_TABLENAME
        USING V_MKID, 0, 1; --主表
      EXECUTE IMMEDIATE v_COL_SQL
        INTO V_TABLENAMEID
        USING V_MKID, 0, 1, 1; --主键Id
      EXECUTE IMMEDIATE v_COL_SQL
        INTO V_TABLENAMELB
        USING V_MKID, 0, 1, 14; --当前类别
      EXECUTE IMMEDIATE v_COL_SQL
        INTO V_TABLENAMEQCLB
        USING V_MKID, 0, 1, 110; --起初类别
      V_SQL := 'UPDATE ' || V_TABLENAME || ' SET ' || V_TABLENAMELB || '=' ||
               V_DLBID || ',' || V_TABLENAMEQCLB || '=' || V_DLBID ||
               ' WHERE ' || V_TABLENAMEID || '=''' || V_DJID || '''';
      EXECUTE IMMEDIATE v_SQL;
    END IF;
  END IF;
END;
```

### 5.4 BLOB

```sql
--blob查询
select * from qrtz_job_details_local t 
where dbms_lob.instr(job_data,utl_raw.cast_to_raw('declarano'),1,1)<>0;
```

### 5.5 Shell调试

有入参的返回游标过程 vi proc-debug.sh
```shell
#!/bin/bash
starttime=`date +'%Y-%m-%d %H:%M:%S'`
echo '开始执行时间-'$starttime >> /data/kh_shell/proc_debug.log

su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
var a1 VARCHAR2;
var a2 VARCHAR2;
var V3 refcursor;
CALL PROC_WZCHECK_COUNT_BY_MONTH(:a1,:a2,:V3);
commit;
disconnect
quit
EOF"

etime=`date +'%Y-%m-%d %H:%M:%S'`
echo 'PROC_WZCHECK_COUNT_BY_MONTH完成时间-'$etime >> /data/kh_shell/proc_debug.log
```

```sql
CREATE OR REPLACE PROCEDURE PROC_WZCHECK_COUNT_BY_MONTH(USERID    OUT VARCHAR2,
                                                   USERTYPE  OUT VARCHAR2,
                                                   V_OUTLIST OUT SYS_REFCURSOR) AS
  V_STR VARCHAR2(800);
  V_DATE    DATE;
  V_NUM    NUMBER; ---回访数
BEGIN
  OPEN V_OUTLIST FOR
    SELECT * FROM EMP;
END;
```

```bash
nohup sh /data/proc-debug.sh &     # 后台执行
```