# Keepalived 2.0.20

Keepalived是一款由C编写的软件，一般解决负载均衡器的高可用性问题，提供了负载均衡、健康检查和高可用的功能，高可用功能是由VRRP协议来实现的

- 官方网站：https://keepalived.org

## 1. 安装

### 1.1 安装依赖

```bash
yum install -y gcc openssl-devel popt-devel
```

### 1.2 解压编译

```bash
cd /opt/software
tar -zxf keepalived-2.0.20.tar.gz -C /opt/
cd /opt/keepalived-2.0.20
./configure --sysconf=/etc --prefix=/usr/local/keepalived
make && make install
```

### 1.3 修改配置


#### 1.3.1 备份默认配置

```bash
cd /etc/keepalived/
cp keepalived.conf keepalived.conf.default
```

#### 1.3.2 配置说明

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
    script "/etc/keepalived/nginx_check.sh"     # 检测nginx状态的脚本路径
    interval 2                                  # 检测时间间隔
    weight -20                                  # 如果条件成立，权重-20
}

vrrp_sync_group {

}

garp_group {

}

# 设置keepalived实例的相关信息，VI_1为VRRP实例名称
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

### 1.4 启动服务

```bash
cd /usr/local/keepalived/sbin
./keepalived
tail -f /var/log/messages
ip a
```

## 2. 高可用

### 2.1 nginx

#### 2.1.1 主节点

```conf
global_defs {
    router_id keep129;
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 129
    mcast_src_ip 192.168.211.129
    priority 100
    nopreempt
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    
    track_script {
        chk_nginx
    }
    virtual_ipaddress {
        192.168.211.131
    }
}
```

#### 2.1.2 备用节点

```conf
global_defs {
    router_id keep130
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth2
    virtual_router_id 130
    priority 90
    mcast_src_ip 192.168.211.130
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script {
        chk_nginx
    }
    virtual_ipaddress {
        192.168.211.131
    }
}
```

#### 2.1.3 检测脚本


添加文件位置为/etc/keepalived/nginx_check.sh

```bash
#!/bin/bash
A=`ps -C nginx –no-header |wc -l`
if [ $A -eq 0 ];then
    /usr/local/server/nginx/sbin/nginx
    sleep 2
    if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
        killall keepalived
    fi
fi

```

```bash
chmod 755 nginx_check.sh
```

### 2.2 haproxy

#### 2.2.1 主节点

```conf
global_defs {
    router_id LVS_R1
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/haproxy_check.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 88
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.16.8
    }
    track_script {
        check_haproxy
    }
}
```

#### 2.2.2 备用节点

```conf
global_defs {
    router_id LVS_R2
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/haproxy_check.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 88
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.16.8
    }
    track_script {
        check_haproxy
    }
}
```

#### 2.2.3 检测脚本

添加文件位置为/etc/keepalived/haproxy_check.sh

```shell
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
sed -i 's/\r$//' haproxy_check.sh
```