# Zabbix 6.0 LTS for Rocky Linux 8, MySQL, NGINX

https://www.zabbix.com/download

## 1. 安装 Zabbix Server

### 1.1 一键安装

```bash
#!/bin/bash
#
#********************************************************************
#Zabbix 6.0 LTS for Rocky Linux 8, MySQL, NGINX
#********************************************************************

ZABBIX_VER=6.0
URL="mirror.tuna.tsinghua.edu.cn/zabbix"

MYSQL_HOST=localhost
MYSQL_ZABBIX_USER="zabbix@localhost"

MYSQL_ZABBIX_PASS='123456'
MYSQL_ROOT_PASS='123456'

FONT=msyhbd.ttc

ZABBIX_IP=`hostname -I|awk '{print $1}'`
GREEN="echo -e \E[32;1m"
END="\E[0m"

color () {
    RES_COL=60
    MOVE_TO_COL="echo -en \\033[${RES_COL}G"
    SETCOLOR_SUCCESS="echo -en \\033[1;32m"
    SETCOLOR_FAILURE="echo -en \\033[1;31m"
    SETCOLOR_WARNING="echo -en \\033[1;33m"
    SETCOLOR_NORMAL="echo -en \E[0m"
    echo -n "$1" && $MOVE_TO_COL
    echo -n "["
    if [ $2 = "success" -o $2 = "0" ] ;then
        ${SETCOLOR_SUCCESS}
        echo -n $"  OK  "    
    elif [ $2 = "failure" -o $2 = "1"  ] ;then 
        ${SETCOLOR_FAILURE}
        echo -n $"FAILED"
    else
        ${SETCOLOR_WARNING}
        echo -n $"WARNING"
    fi
    ${SETCOLOR_NORMAL}
    echo -n "]"
    echo 
}

install_mysql () {
    yum -y install mysql-server
    systemctl enable --now mysqld
    mysqladmin -uroot password $MYSQL_ROOT_PASS
    mysql -uroot -p$MYSQL_ROOT_PASS <<EOF
create database zabbix character set utf8 collate utf8_bin;
create user $MYSQL_ZABBIX_USER identified by "$MYSQL_ZABBIX_PASS";
grant all privileges on zabbix.* to $MYSQL_ZABBIX_USER;
quit
EOF
    if [ $? -eq 0 ];then
        color "MySQL数据库准备完成" 0
    else
        color "MySQL数据库配置失败,退出" 1
        exit
    fi
}

install_zabbix () {
	rpm -Uvh https://${URL}/zabbix/${ZABBIX_VER}/rhel/8/x86_64/zabbix-release-${ZABBIX_VER}-1.el8.noarch.rpm
	if [ $? -eq 0 ];then
		color "YUM仓库准备完成" 0
	else
		color "YUM仓库配置失败,退出" 1
		exit
	fi
	sed -i "s#repo.zabbix.com#${URL}#" /etc/yum.repos.d/zabbix.repo
	yum -y install zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent zabbix-get langpacks-zh_CN

}
config_mysql_zabbix () {
	if [ -f "$FONT" ] ;then 
	    mv /usr/share/zabbix/assets/fonts/graphfont.ttf{,.bak}
		cp  "$FONT" /usr/share/zabbix/assets/fonts/graphfont.ttf
	else
		color "缺少字体文件!" 1
	fi
	if [ $MYSQL_HOST = "localhost" ];then
		zcat /usr/share/doc/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -p$MYSQL_ZABBIX_PASS -h$MYSQL_HOST zabbix
	fi
	sed -i -e "/.*DBPassword=.*/c DBPassword=$MYSQL_ZABBIX_PASS" -e "/.*DBHost=.*/c DBHost=$MYSQL_HOST" /etc/zabbix/zabbix_server.conf
	sed -i -e "/.*date.timezone.*/c php_value[date.timezone] = Asia/Shanghai" -e "/.*upload_max_filesize.*/c php_value[upload_max_filesize] = 20M" /etc/php-fpm.d/zabbix.conf
	systemctl enable --now zabbix-server zabbix-agent httpd php-fpm

    if [ $?  -eq 0 ];then  
        echo 
        color "ZABBIX-${ZABBIX_VER}安装完成!" 0
        echo "-------------------------------------------------------------------"
        ${GREEN}"请访问: http://$ZABBIX_IP/zabbix"${END}
    else
        color "ZABBIX-${ZABBIX_VER}安装失败!" 1
        exit
    fi
}

install_mysql
install_zabbix
config_mysql_zabbix
```

### 1.2 启动


### 1.3 卸载

```bash
pkill -9 zabbix
yum -y remove mysql-server
yum -y remove zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent zabbix-get langpacks-zh_CN
rpm -e zabbix-release-6.0-1.el8.noarch --nodeps

rm -rf `find / -name zabbix` 
rm -rf `find / -name mysql` 
rm -rf /etc/my.cnf
```


## 2. 安装 Zabbix Agent

### 2.1 一键安装

### 2.2 启动

```bash
ss -ntlp|grep zabbix    # 检测是否安装成功
```

### 2.3 卸载

```bash
yum -y remove zabbix-agent2
```