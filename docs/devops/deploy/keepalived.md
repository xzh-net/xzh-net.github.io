# Keepalived 2.0.20

## 1. 安装

https://keepalived.org/

```bash
yum install -y gcc openssl-devel popt-devel # 安装依赖
cd ~
mkdir keepalived
tar -zxf keepalived-2.0.20.tar.gz -C keepalived/
cd keepalived/keepalived-2.0.20
./configure --sysconf=/etc --prefix=/usr/local/keepalived
make && make install

# 备份默认配置文件
cd /etc/keepalived/
cp keepalived.conf keepalived.conf.default
# 启动
cd /usr/local/keepalived/sbin
./keepalived
tail -f /var/log/messages
ip a
```

## 2. 高可用

### 2.1 nginx

1. 配置说明

```conf
global_defs {
    notification_email { 
        xcg992224@163.com                       # 通知邮件，当keepalived发送切换时需要发email给具体的邮箱地址
    }
    notification_email_from admin@xuzhihao.net  # 设置发件人的邮箱信息
    smtp_server 192.168.200.1                   # 指定smpt服务地址
    smtp_connect_timeout 30                     # 指定smpt服务连接超时时间
    router_id LVS_DEVEL                         # 运行keepalived服务器的一个标识，可以用作发送邮件的主题信息
    vrrp_skip_check_adv_addr
    vrrp_strict                                 # 严格遵守VRRP协议。
    vrrp_garp_interval 0                        # 在一个接口发送的两个免费ARP之间的延迟。可以精确到毫秒级。默认是0
    vrrp_gna_interval 0                         # 在一个网卡上每组na消息之间的延迟时间，默认为0
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh" # 检测nginx状态的脚本路径
    interval 2                              # 检测时间间隔
    weight -20                              # 如果条件成立，权重-20
}

vrrp_sync_group {

}

garp_group {

}

#设置keepalived实例的相关信息，VI_1为VRRP实例名称
vrrp_instance VI_1 {
    state MASTER          # 主节点为MASTER， 备份节点为BACKUP
    interface enp0s3      # vrrp实例绑定网卡名称
    virtual_router_id 51  # 指定VRRP实例ID，范围是0-255
    priority 100          # 指定优先级，优先级高的将成为MASTER
    nopreempt             # 优先级高的设置 nopreempt解决异常恢复后再次抢占的问题
    advert_int 1          # 组播信息发送间隔，两个节点设置必须一样， 默认1s
    authentication {      # 设置验证信息，两个节点必须一致
        auth_type PASS    # 指定认证方式。PASS简单密码认证(推荐)
        auth_pass 1111    # 指定认证使用的密码，最多8位
    }
    virtual_ipaddress {   # 虚拟IP池, 两个节点设置必须一样，可多个
        192.168.200.222
    }
    track_script {
      chk_nginx
    }
}
```

2. 主节点


```conf
global_defs {
	router_id LVS_DEVEL
}

vrrp_instance VI_1 {
	state MASTER
	interface enp0s3
	virtual_router_id 51
	priority 100
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	virtual_ipaddress {
		172.16.10.113
	}
}
```

3. 备用节点

```conf
global_defs {
	router_id LVS_DEVEL
}

vrrp_instance VI_1 {
	state BACKUP 
	interface enp0s3
	virtual_router_id 51
	priority 90
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	virtual_ipaddress {
		172.16.10.113
	}
}
```

4. 检测脚本

nginx
```bash
#!/bin/bash
num=`ps -C nginx --no-header | wc -l`
if [ $num -eq 0 ];then
 /usr/local/nginx/sbin/nginx
 sleep 2
 if [ `ps -C nginx --no-header | wc -l` -eq 0 ]; then
  killall keepalived
 fi
fi
```

```bash
chmod 755 ck_nginx.sh
```

### 2.2 haproxy

1. 主节点

```conf
global_defs {
	notification_email {
		javadct@163.com
	}
	notification_email_from keepalived@showjoy.com
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	router_id haproxy01
	vrrp_skip_check_adv_addr
	vrrp_garp_interval 0
	vrrp_gna_interval 0
}

vrrp_script chk_haproxy {
	script "/etc/keepalived/haproxy_check.sh"
	interval 2
	weight 2
}

vrrp_instance VI_1 {
	#主机配MASTER，备机配BACKUP
	state MASTER
	#所在机器网卡
	interface eth1
	virtual_router_id 51
	#数值越大优先级越高
	priority 120
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	## 将 track_script 块加入 instance 配置块
    track_script {
    	chk_haproxy ## 检查 HAProxy 服务是否存活
    }
	virtual_ipaddress {
		#虚拟IP
		192.168.192.200
	}
}
```

2. 备用节点

```conf
global_defs {
	notification_email {
		javadct@163.com
	}
	notification_email_from keepalived@showjoy.com
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	router_id haproxy02	# 标识备节点
	vrrp_skip_check_adv_addr
	vrrp_garp_interval 0
	vrrp_gna_interval 0
}

# keepalived 会定时执行脚本并对脚本执行的结果进行分析，动态调整 vrrp_instance 的优先级
vrrp_script chk_haproxy {
	script "/etc/keepalived/haproxy_check.sh"	# 检测 haproxy 状态的脚本路径
	interval 2									# 检测时间间隔
	weight 2									# 如果条件成立，权重+2
}

vrrp_instance VI_1 {
	#主机配MASTER，备机配BACKUP
	state BACKUP
	#所在机器网卡
	interface eth1
	virtual_router_id 51
	#数值越大优先级越高
	priority 100
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	## 将 track_script 块加入 instance 配置块
    track_script {
    	chk_haproxy ## 检查 HAProxy 服务是否存活
    }
	virtual_ipaddress {
		#虚拟IP
		192.168.192.200
	}
}
```

3. 检测脚本haproxy_check.sh

```shell
#添加文件位置为/etc/keepalived/haproxy_check.sh
#!/bin/bash
COUNT=`ps -C haproxy --no-header |wc -l`
if [ $COUNT -eq 0 ];then
    /usr/local/haproxy/sbin/haproxy -f /etc/haproxy/haproxy.cfg
    sleep 2
    if [ `ps -C haproxy --no-header |wc -l` -eq 0 ];then
        killall keepalived
    fi
fi
```

```bash
chmod +x /etc/keepalived/haproxy_check.sh
sed -i 's/\r$//' check_haproxy.sh
```