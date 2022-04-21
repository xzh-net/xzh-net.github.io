# yum安装Zabbix

## 1. 准备工作
 - 服务器配置固定的IP地址和DNS
 - 实验环境:
 - 操作系统安装与VMware
 - CPU：Intel® Xeon® CPU E5-2660 v3
 - Mem: 8GB
 - Disk: 100GB
 - 操作系统：CentOS 7.8 x86_64


### 1.1 关闭防火墙和SELinux
安装配置时关闭防火墙，避免一些坑，服务调试完成，没有问题时再开启防火墙添加相关服务通过防火墙
```bash
systemctl stop firewalld
systemctl disable firewalld
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config 
setenforce 0
```

### 1.2 添加本地DNS解析
如果不添加，后面前端页面会有一些报错。
```bash
vim /etc/hosts
192.168.3.200 monitor01.smartlinux.cn monitor01
```

### 1.3 配置yum源
CentOS默认的yum源是使用国外的服务器，修改为国内的阿里yum源
```bash
mkdir /etc/yum.repos.d/repo.bak
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/repo.bak
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```
添加Zabbix源，鉴于国内网络情况，使用阿里云 zabbix 
```bash
rpm -Uvh https://mirrors.aliyun.com/zabbix/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
sed -i 's#http://repo.zabbix.com#https://mirrors.aliyun.com/zabbix#' /etc/yum.repos.d/zabbix.repo
```
安装 Software Collections，便于后续安装高版本的 php，默认 yum 安装的 php 版本为 5.4 。
```bash
yum install -y centos-release-scl
```
启用 Zabbix 前端源，修改/etc/yum.repos.d/zabbix.repo,将[zabbix-frontend]下的 enabled 改为 1
```bash
[zabbix-frontend]
enabled=1
sed -i '11 s/0/1/' /etc/yum.repos.d/zabbix.repo
```

## 2. 安装 Zabbix
### 2.1 安装zabbix-server和agent

```bash
yum install -y zabbix-server-mysql zabbix-agent
```
### 2.2 安装Zabbix前端和相关环境

```bash
yum install -y zabbix-web-mysql-scl zabbix-apache-conf-scl
```
### 2.3 安装centos7默认的mariadb数据库

```bash
yum install -y mariadb-server
```
## 3. 配置zabbix
### 3.1 配置mariadb数据库

启动数据库，并配置开机自动启动

```bash
systemctl enable --now mariadb
mysqladmin -uroot password "mysql123"
```
使用 root 用户进入 mysql，并建立 zabbix 数据库，注意数据库编码。数据库名称为zabbix
```sql
mysql -uroot -p
create database zabbixdb character set utf8 collate utf8_bin;
grant all privileges on zabbixdb.* to zabbixuser@localhost identified by 'zabbix9527';
grant all privileges on zabbixdb.* to zabbixuser@'192.168.3.%' identified by 'zabbix51769';
grant all privileges on zabbixdb.* to zabbixuser@'monitor01.smartlinuxs.cn' identified by 'zabbix51769';
flush privileges;
quit;
```
使用以下命令导入 zabbix 数据库，zabbix 数据库用户为 zabbixuser，密码为 zabbix9527
```bash
zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbixuser -pzabbix9527 zabbixdb
```
### 3.2 配置zabbix server数据库连接 

修改 zabbix server 配置文件/etc/zabbix/zabbix_server.conf 里的数据库密码

```bash
vim /etc/zabbix/zabbix_server.conf
DBHost=192.168.3.200
DBName=zabbixdb
DBUser=zabbixuser
DBPassword=zabbix9527
StartDiscoverers=2

sed -i '91a DBHost=192.168.8.139.200' /etc/zabbix/zabbix_server.conf
sed -i '101 s/zabbix/zabbixdb/' /etc/zabbix/zabbix_server.conf
sed -i '117 s/zabbix/zabbixuser/' /etc/zabbix/zabbix_server.conf
sed -i '126a DBPassword=zabbix51769' /etc/zabbix/zabbix_server.conf
sed -i '246a StartDiscoverers=2' /etc/zabbix/zabbix_server.conf
```
### 3.4 修改 zabbix 的 php 配置文件

修改 /etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf 里的时区

```bash
php_value[date.timezone] = Asia/Shanghai
sed -i /timezone/d /etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf
echo 'php_value[date.timezone] = Asia/Shanghai'>>/etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf
```
### 3.5 启动相关服务，并配置开机自动启动

```bash
systemctl restart zabbix-server zabbix-agent httpd rh-php72-php-fpm
systemctl enable zabbix-server zabbix-agent httpd rh-php72-php-fpm
netstat -lntp|grep -E "zabbix|http|mysql|php"
```
## 4. 前端页面配置

### 4.1 启动相关服务，并配置开机自动启动

- 使用浏览器访问 http://ip/zabbix 即可访问 zabbix 的 web 页面
- 默认账号：Admin
- 默认密码：zabbix

### 4.2 自动配置前端页面
不喜欢下一步，下一步的点，可以直接创建这个php的页面
```bash
cat >/etc/zabbix/web/zabbix.conf.php<<- 'EOC'
<?php
// Zabbix GUI configuration file.

$DB['TYPE'] = 'MYSQL';
$DB['SERVER'] = '192.168.3.200';
$DB['PORT'] = '3306';
$DB['DATABASE'] = 'zabbixdb';
$DB['USER'] = 'zabbixuser';
$DB['PASSWORD'] = 'zabbix51769';

// Schema name. Used for PostgreSQL.
$DB['SCHEMA'] = '';

// Used for TLS connection.
$DB['ENCRYPTION'] = false;
$DB['KEY_FILE'] = '';
$DB['CERT_FILE'] = '';
$DB['CA_FILE'] = '';
$DB['VERIFY_HOST'] = false;
$DB['CIPHER_LIST'] = '';

// Use IEEE754 compatible value range for 64-bit Numeric (float) history values.
// This option is enabled by default for new Zabbix installations.
// For upgraded installations, please read database upgrade notes before enabling this option.
$DB['DOUBLE_IEEE754'] = true;

$ZBX_SERVER = 'monitor01.smartlinux.cn';
$ZBX_SERVER_PORT = '10051';
$ZBX_SERVER_NAME = 'monitor01';

$IMAGE_FORMAT_DEFAULT = IMAGE_FORMAT_PNG;

// Uncomment this block only if you are using Elasticsearch.
// Elasticsearch url (can be string if same url is used for all types).
//$HISTORY['url'] = [
// 'uint' => 'http://localhost:9200',
// 'text' => 'http://localhost:9200'
//];
// Value types stored in Elasticsearch.
//$HISTORY['types'] = ['uint', 'text'];

// Used for SAML authentication.
// Uncomment to override the default paths to SP private key, SP and IdP X.509 certificates, and to set extra settings.
//$SSO['SP_KEY'] = 'conf/certs/sp.key';
//$SSO['SP_CERT'] = 'conf/certs/sp.crt';
//$SSO['IDP_CERT'] = 'conf/certs/idp.crt';
//$SSO['SETTINGS'] = [];
EOC

#修改文件的属主
chown apache:apache /etc/zabbix/web/zabbix.conf.php
```
### 4.3 安装中文字体

默认使用的字体不包含中文，所以web页面中部分中文显示会乱码。

```bash
/usr/share/zabbix/assets/fonts/graphfont.ttf #默认使用的字体，实际上这是个软连接。
```
安装新的中文字体
```bash
yum install -y wqy-microhei-fonts
rm -f /usr/share/zabbix/assets/fonts/graphfont.ttf
ln -s /usr/share/fonts/wqy-microhei/wqy-microhei.ttc /usr/share/zabbix/assets/fonts/graphfont.ttf
```
删除默认字体的链接，重新创建一个链接。
```bash
rm -f /usr/share/zabbix/assets/fonts/graphfont.ttf
ln -s /usr/share/fonts/wqy-microhei/wqy-microhei.ttc /usr/share/zabbix/assets/fonts/graphfont.ttf
```
此时查看zabbix的web界面，中文已经正常显示了。