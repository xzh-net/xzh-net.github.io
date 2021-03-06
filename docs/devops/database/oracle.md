# Oracle 11g R2

## 1. 安装

### 1.1 静默安装11.2.0.1.0

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
    - 32位linux系统：可取最大值为 4GB （ 4294967296bytes ） -1byte ，即 4294967295 。建议值为多于内存的一半，所以如果是 32 为系统，一般可取值为 4294967295 。 32 位系统对 SGA 大小有限制，所以 SGA 肯定可以包含在单个共享内存段中。
    - 64位linux系统：可取的最大值为物理内存值 -1byte ，建议值为多于物理内存的一半，一般取值大于 SGA_MAX_SIZE 即可，可以取物理内存 -1byte 。  

  ?> 内存为 12G 时，该值为 12x1024x1024x1024-1 = 12884901887

vim /etc/sysctl.conf
```bash
fs.aio-max-nr=1048576
fs.file-max=6815744
kernel.shmall=2097152
kernel.shmmni=4096
kernel.shmmax = 8589934591
kernel.sem=250 32000 100 128
net.ipv4.ip_local_port_range=9000 65500
net.core.rmem_default=262144
net.core.rmem_max=4194304
net.core.wmem_default=262144
net.core.wmem_max=1048586
# 使内核新配置生效
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
!> 等待...[WARING]可暂时忽略，此时安装程序仍在后台进行，如果出现[FATAL]，则安装程序已经异常停止了,当出现 Successfully Setup Software. 证明已经安装成功，然后根据提示操作
```lua
正在启动 Oracle Universal Installer...

检查临时空间: 必须大于 120 MB。   实际为 41029 MB    通过
检查交换空间: 必须大于 150 MB。   实际为 3967 MB    通过
...
 /u01/app/oracle/inventory/logs/installActions2022-06-09_04-14-01PM.log
以下配置脚本需要以 "root" 用户的身份执行。
 #!/bin/sh 
 #要运行的 Root 脚本

/u01/app/oracle/inventory/orainstRoot.sh
/u01/app/oracle/product/11.2.0/dbhome_1/root.sh
要执行配置脚本, 请执行以下操作:
	 1. 打开一个终端窗口
	 2. 以 "root" 身份登录
	 3. 运行脚本
	 4. 返回此窗口并按 "Enter" 键继续

Successfully Setup Software.

```

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
成功运行后，在 `/u01/app/oracle/product/11.2.0/dbhome_1/network/admin/` 中生成`listener.ora`和`sqlnet.ora`
```bash
cat $ORACLE_HOME/network/admin/listener.ora     # 查看监听器配置文件
cat $ORACLE_HOME/network/admin/sqlnet.ora       # 查看监听服务名配置文件
yum install net-tools
netstat -tunlp|grep 1521
```


#### 1.1.15 以静默方式建立新库，同时也建立一个对应的实例

```bash
su - oracle
egrep -v "(^#|^$)" /home/oracle/response/dbca.rsp # 查看建库相应文件配置信息
vim /home/oracle/response/dbca.rsp # 编辑应答文件

# 设置CREATEDATABASE以下参数
GDBNAME = "orcl"
SID = "orcl"
DATAFILEDESTINATION =/u01/app/oracle/oradata
RECOVERYAREADESTINATION=/u01/app/oracle/fast_recovery_area
CHARACTERSET = "AL32UTF8"
TOTALMEMORY = "4096"
```

静默配置
```bash
dbca -silent -responseFile /home/oracle/response/dbca.rsp
```
执行完后会先清屏，清屏之后没有提示，直接输入oracle用户的密码，回车，再输入一次，再回车。稍等一会，会开始自动创建，建库后进行实例进程检查 
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


### 1.2 图形化安装11.2.0.4.0

#### 1.2.1 一键安装和配置VNC图形化相关

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

![](../../assets/_images/devops/database/oracle/step1.png)

跳过软件更新

![](../../assets/_images/devops/database/oracle/step2.png)

创建和配置数据库

![](../../assets/_images/devops/database/oracle/step3.png)

选择服务器类

![](../../assets/_images/devops/database/oracle/step4.png)

默认选择单实例数据库

![](../../assets/_images/devops/database/oracle/step5.png)

默认典型安装

![](../../assets/_images/devops/database/oracle/step6.png)

设置实例和密码，其他默认即可。这里密码要在大写字母+小写字母+数字组合。比如：1234Qwer

![](../../assets/_images/devops/database/oracle/step7.png)

创建产品清单，默认

![](../../assets/_images/devops/database/oracle/step8.png)

执行先决条件检查，按照静默方式的内核参数和限制设定

```bash
# 创建swap文件 bs=2300的设置的值一般为内存的1.5倍以上 
dd if=/dev/zero of=swapfree bs=32k count=65515
mkswap swapfree # 将创建的文件用做交换分区
swapon swapfree # 开启交换分区
# 使Swap文件永久生效,/etc/fstab加入配置
echo "swapfree   swap    swap    sw  0   0" >> /etc/fstab
```

![](../../assets/_images/devops/database/oracle/step9.png)

准备安装

![](../../assets/_images/devops/database/oracle/step10.png)

进行安装

![](../../assets/_images/devops/database/oracle/step11.png)


进度70% `ins_emagent.mk错误弹框`

![](../../assets/_images/devops/database/oracle/step11_error.png)


?> 编辑 /home/oracle/app/oracle/product/11.2.0/dbhome_1/sysman/lib/ins_emagent.mk 约176行，可以搜索$(MK_EMAGENT_NMECTL) 关键字快速找到。

![](../../assets/_images/devops/database/oracle/step11_update.png)

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

![](../../assets/_images/devops/database/oracle/step11_goon.png)

数据库创建完成

![](../../assets/_images/devops/database/oracle/step11_done.png)

执行配置脚本

![](../../assets/_images/devops/database/oracle/step11_shell.png)

使用root身份执行

```bash
/home/oracle/app/oraInventory/orainstRoot.sh 
/home/oracle/app/oracle/product/11.2.0/dbhome_1/root.sh 
```

执行完成这两个脚本，点击确定

![](../../assets/_images/devops/database/oracle/step11_shell_ok.png)

Oracle Database 安装成功

![](../../assets/_images/devops/database/oracle/step12.png)

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

### 1.3 RAC

1. 网络配置
2. 主机及host配置
3. 生成共享磁盘
4. RAC_ASM磁盘挂载
5. 用户变量及互信
6. 集群安装
7. 安装数据库
8. 建库

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

### 2.1 重启

#### 2.1.1 命令重启

```sql
su – oracle
sqlplus /nolog
connect / as sysdba
shutdown immediate
startup
```

#### 2.1.2 脚本重启

vi /data/oracle_restart.sh
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

### 2.2 表空间

#### 2.2.1 临时表空间
  
表空间名字不能重复，即便存储的位置不一致, 但是dbf文件可以一致，50m为表空间的大小，对大数据量建议32G

```bash
create temporary tablespace xzh_temp
tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' 
size 50m
autoextend on 
next 50m maxsize 20480m 
extent management local;
```

```bash
select tablespace_name,file_name,bytes/1024/1024 file_size,autoextensible from dba_temp_files;        # 查询临时表空间
create temporary tablespace xzh_temp tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' size 10M;        # 创建临时表空间
alter database tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' resize 100M;                           # 调整临时表空间大小
alter tablespace xzh_temp add tempfile '/u01/app/oracle/oradata/xzh_temp_2.dbf' size 100m;              # 向临时表空间中添加数据文件： 
alter database tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' autoextend on next 5m maxsize unlimited; # 将临时数据文件设为自动扩展
alter database tempfile '/u01/app/oracle/oradata/xzh_temp.dbf' drop;                                    # 删除临时表空间的一个数据文件
drop tablespace xzh_temp including contents and datafiles cascade constraints;                          # 删除临时表空间(彻底删除)
```

#### 2.2.2 数据表空间

```bash
create tablespace xzh
logging 
datafile '/u01/app/oracle/oradata/xzh.dbf' 
size 50m 
autoextend on 
next 50m maxsize 20480m 
extent management local;
```

```bash
select tablespace_name, sum(bytes)/1024/1024 from dba_data_files group by tablespace_name;            # 查询表空间
create tablespace xuzhihao datafile '/u01/app/oracle/xuzhihao/xuzhihao.dbf' size 100M;                # 创建表空间
alter tablespace xuzhihao add datafile '/u01/app/oracle/xuzhihao/xuzhihao.dbf' size 5m;               # 增加表空间数据文件
drop tablespace xuzhihao including contents;                                                          # 删除表空间
drop tablespace xuzhihao including contents and datafiles;                                            # 删除表空间和数据文件
alter database datafile '/u01/app/oracle/xuzhihao/xuzhihao.dbf' resize 50m;                           # 扩展表空间
alter database datafile '/u01/app/oracle/xuzhihao/xuzhihao.dbf' autoextend on next 50m maxsize 500m;  # 表空间自动增长
```

### 2.3 创建用户授权

```sql
create user xzh0610 identified by 123456
  default tablespace xzh
  temporary tablespace xzh_temp
  profile DEFAULT
  password  expire;

grant connect,resource to xzh0610;
grant read,write ON DIRECTORY oradmp to xzh0610; 
grant dba to xzh0610;
drop user xzh0610 cascade;
```


### 2.4 目录管理

```sql
SELECT * FROM DBA_DIRECTORIES;                          --查看目录
CREATE DIRECTORY oradmp AS '/u01/app/oracle/oradmp';    -- 创建目录
DROP DIRECTORY oradmp;                                  -- 删除目录
GRANT READ,WRITE ON DIRECTORY oradmp to xzh0610;        --将oradmp目录的赋给用户
```

### 2.5 备份恢复

#### 2.5.1 按表名备份、还原

```bash
expdp xzh0610/123456 directory=oradmp dumpfile=xzh0610.dmp tables=sys_menu,sys_role,sys_user  
impdp xzh0610/123456 directory=oradmp dumpfile=xzh0610.dmp tables=xzh0610.sys_menu,xzh0610.sys_user REMAP_SCHEMA=xzh0610:xzh0610 table_exists_action=replace
```

#### 2.5.2 全量备份、还原

```bash
expdp xzh0610/123456 directory=oradmp dumpfile=xzh0610.dmp SCHEMAS=xzh0610 logfile=xzh0610_$(date +%Y%m%d-%H%M).log
impdp xzh0611/123456 directory=oradmp dumpfile=xzh0610.dmp  schemas=xzh0610 REMAP_SCHEMA=xzh0610:xzh0611 REMAP_TABLESPACE=xzh:xzh

# 还原后无法使用
execute dbms_stats.delete_schema_stats('xzh0610');
```

#### 2.5.3 定时数据还原

```bash
crontab -e
30 3 * * * sh /data/shell/ei.sh  # 每天凌晨3点半执行
```

vi /data/shell/ei.sh
```sh
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
echo '还原数据库5开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "impdp VJSP_JSWZ_190601/wzld9999 directory=KH_DIR dumpfile=VJSP_JSWZ_${BAKDATE}_%U.dmp parallel=4 logfile=5_${BAKDATE}.log remap_schema=VJSP_JSWZ_190601:VJSP_JSWZ_190601 TABLE_EXISTS_ACTION=TRUNCATE tables=TS_SYSTEM_DYWJ"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '还原数据库5结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 5s;
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '还原数据库6开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "impdp VJSP_JSWZ_190601/wzld9999 directory=KH_DIR dumpfile=VJSP_JSWZ_${BAKDATE}_%U.dmp parallel=4 logfile=6_${BAKDATE}.log remap_schema=VJSP_JSWZ_190601:VJSP_JSWZ_190601 TABLE_EXISTS_ACTION=TRUNCATE tables=TS_FLOW_MAIN_MX"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '还原数据库6结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 5s;
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '还原数据库7开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "impdp VJSP_JSWZ_190601/wzld9999 directory=KH_DIR dumpfile=VJSP_JSWZ_${BAKDATE}_%U.dmp parallel=4 logfile=7_${BAKDATE}.log remap_schema=VJSP_JSWZ_190601:VJSP_JSWZ_190601 TABLE_EXISTS_ACTION=TRUNCATE tables=TS_FLOW_PATH_COM"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '还原数据库7结束时间：'$etime >> /data/kh_shell/kh_log.log

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
echo '脚本3开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
call PROC_DEL_ONEYEAR();
commit;
call PROC_UPDATECFID0();
commit;
call proc_up_path2('');
commit;
call PROC_GET_DEALTIME('',to_char(TRUNC(SYSDATE - 1, 'MM'),'YYYY/MM/DD HH24:MI:SS'),to_char(TRUNC(LAST_DAY(sysdate-1) + 1, 'MM'),'YYYY/MM/DD HH24:MI:SS'));
commit;
disconnect
quit
EOF"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '脚本3结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 5s;
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '脚本4开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
call PROC_GET_NOTME('','');
commit;
disconnect
quit
EOF"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '脚本4结束时间：'$etime >> /data/kh_shell/kh_log.log

sleep 5s;
stime=`date +'%Y-%m-%d %H:%M:%S'`
echo '提交事务开始时间：'$stime >> /data/kh_shell/kh_log.log
su - oracle -c "sqlplus VJSP_JSWZ_190601/wzld9999<< EOF
commit;
disconnect
quit
EOF"
etime=`date +'%Y-%m-%d %H:%M:%S'`
echo '提交事务结束时间：'$etime >> /data/kh_shell/kh_log.log

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

### 2.6 生成AWR报告

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


## 3. 表操作

### 3.1 系统参数

```sql
SELECT version FROM product_component_version WHERE substr(product, 1, 6) = 'Oracle';         -- 查看版本
SELECT created, log_mode, log_mode FROM v$database;                                           -- 查看归档方式
select username,count(username) from v$session where username is not null group by username;  -- 查看不同用户的连接数
select count(*) from v$session where status='ACTIVE';                                         -- 查询oracle的并发连接数
select a.*,round(a.bytes/1024/1024,2) M from v$sgastat a where a.NAME = 'free memory';        -- 查询share pool的空闲内存
select count(*) from v$session;                          -- 查看数据库当前链接数
SELECT NAME FROM v$controlfile;                          -- 查看控制文件 
select value from v$parameter where name = 'processes';  -- 最大连接
alter system set processes = value scope = spfile;       -- 修改连接数需重启

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

-- alter失败执行ps -ef|grep spid
select pro.spid from v$session ses,v$process pro where ses.sid={sid} and ses.paddr=pro.addr;
```

### 3.3 等待事件

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

-- iotop
SELECT s.sql_text FROM v$sql s, v$session t,v$process v WHERE s.sql_id = t.SQL_ID AND t.PADDR = v.ADDR AND v.SPID = '44196';

SELECT S.SADDR, S.SID, S.SERIAL#, S.MACHINE, S.LOGON_TIME  FROM V$SESSION S
 WHERE PADDR IN (SELECT ADDR FROM V$PROCESS WHERE SPID IN (55751,15842));
```

### 3.4 统计

#### 3.4.1 内存查询

1. SGA/PGA使用率

```sql
SELECT
	name,
	total,
	round( total - free, 2 ) used,
	round( free, 2 ) free,
	round( ( total - free ) / total * 100, 2 ) pctused 
FROM
	(
	SELECT
		'SGA' name,
		( SELECT sum( value / 1024 / 1024 ) FROM v$sga ) total,
		( SELECT sum( bytes / 1024 / 1024 ) FROM v$sgastat WHERE name = 'free memory' ) free 
	FROM
		dual 
	) UNION
SELECT
	name,
	total,
	round( used, 2 ) used,
	round( total - used, 2 ) free,
	round( used / total * 100, 2 ) pctused 
FROM
	(
	SELECT
		'PGA' name,
		( SELECT value / 1024 / 1024 total FROM v$pgastat WHERE name = 'aggregate PGA target parameter' ) total,
		( SELECT value / 1024 / 1024 used FROM v$pgastat WHERE name = 'total PGA allocated' ) used 
	FROM
	dual 
	);
```

2. 查询占用share pool内存大于10M的sql

```sql
  SELECT substr(sql_text, 1, 100) "Stmt",
         count(*),
         sum(sharable_mem) "Mem",
         sum(users_opening) "Open",
         sum(executions) "Exec"
    FROM v$sql
   GROUP BY substr(sql_text, 1, 100)
  HAVING sum(sharable_mem) > 10000000;
```

3. 查询version count过高SQL

```sql
SELECT address,
       sql_id,
       hash_value,
       version_count,
       users_opening,
       users_executing,
       sql_text
  FROM v$sqlarea
 WHERE version_count > 10;
```

#### 3.4.2 表空间查询

1. 查看当前用户下所有表空间的使用情况

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


#### 3.4.3 数据查询

1. 单表占用物理空间

 ```sql
SELECT SEGMENT_NAME              TABLE_NAME
      ,SUM(BLOCKS)               BLOCKS
      ,SUM(BYTES)/(1024*1024)    "TABLE_SIZE[MB]"
FROM USER_SEGMENTS
WHERE  SEGMENT_TYPE='TABLE'
   AND SEGMENT_NAME='JG_GY_XP'
GROUP BY SEGMENT_NAME;
```

2. 所有表占用物理空间

```sql
SELECT OWNER AS "用户名", SUM(BYTES) / 1024 / 1024 AS "所有表的大小(MB)"
FROM DBA_SEGMENTS
WHERE SEGMENT_NAME IN (SELECT T2.OBJECT_NAME
                        FROM DBA_OBJECTS T2
                       WHERE T2.OBJECT_TYPE = 'TABLE')
GROUP BY OWNER ORDER BY 2 DESC;
```

#### 3.4.4 记录查询

1. 查询某用户下所有表的记录总数

```
SELECT SUM(NUM_ROWS) "记录总条数" FROM SYS.ALL_TABLES T WHERE T.OWNER = 'SHJG0814';
```

2. 查看户下所有表的各自的记录条数

```sql
SELECT T.TABLE_NAME "表名",T.NUM_ROWS "记录条数" FROM USER_TABLES T;
```


## 4. PL/SQL

### 4.1 匿名块

#### 4.1.1 遍历更新

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

#### 4.1.2 DLL遍历更新

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

### 4.2 FUNCTION函数

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

### 4.3 PROCEDURE过程

#### 4.3.1 动态执行SQL

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

### 4.4 BLOB

```sql
--blob查询
select * from qrtz_job_details_local t 
where dbms_lob.instr(job_data,utl_raw.cast_to_raw('declarano'),1,1)<>0;
```

### 4.5 Shell调试

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