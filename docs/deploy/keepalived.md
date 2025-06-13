# Keepalived 2.0.20

Keepalived是一款由C编写的软件，一般解决负载均衡器的高可用性问题，提供了负载均衡、健康检查和高可用的功能，高可用功能是由VRRP协议来实现的

- 官方网站：https://keepalived.org

## 1. 安装

### 1.1 安装依赖

```bash
yum install -y gcc openssl-devel popt-devel libnl libnl-devel
```

### 1.2 解压编译

```bash
cd /opt/software
tar -zxf keepalived-2.0.20.tar.gz
cd keepalived-2.0.20
./configure --sysconf=/etc --prefix=/usr/local/keepalived
make && make install
```

### 1.3 修改配置

```bash
vi /etc/keepalived/keepalived.conf
```

```conf
global_defs {
    router_id ng-01-201;                        # 主机组标识+IP地址
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"     # 检测脚本位置
    interval 10                                 # 检测脚本执行的间隔,秒
    weight -20
}

vrrp_instance VI_1 {
    state MASTER                # 抢占模式： MASTER ，非抢占模式： BACKUP
    interface enp0s3            # 网卡名称
    virtual_router_id 201       # 主、备机的 virtual_router_id 必须相同
    priority 100                # 主、备机取不同的优先级，主机值较大，备份机值较小，一般主100从90
    advert_int 1                # 主、备每隔1秒发送心跳
    nopreempt                   # 非抢占模式，没有主从之分，state全部未BACKUP
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 多网卡下指定广播地址，减少报错和冲突
    unicast_src_ip 192.168.1.31     # 当前主机的IP地址
    unicast_peer {                  # 其他主机的IP地址
        192.168.1.32
    }
    track_script {
        chk_nginx
    }
    virtual_ipaddress {
        192.168.1.201/24
    }
}
```

### 1.4 启动服务

```bash
/usr/local/keepalived/sbin/keepalived
tail -f /var/log/messages
ip a
```

测试vip是否生效

```bash
curl http://192.168.1.201
```

## 2. 配置示例

### 2.1 nginx

这是一个非抢占模式的配置，当主节点宕机后，备份节点会接管vip，但不会自动切换回主节点。

#### 2.1.1 主节点

```conf
global_defs {
    router_id ng-01-201;
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 10
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens3
    virtual_router_id 201
    priority 100
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 多网卡下指定广播地址，减少报错和冲突
    unicast_src_ip 192.168.1.31
    unicast_peer {
        192.168.1.32
    }
    track_script {
        chk_nginx
    }
    virtual_ipaddress {
        192.168.1.201/24
    }
}
```

#### 2.1.2 备用节点

```conf
global_defs {
    router_id ng-01-201;
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 10
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens3
    virtual_router_id 201
    priority 100
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 多网卡下指定广播地址，减少报错和冲突
    unicast_src_ip 192.168.1.32
    unicast_peer {
        192.168.1.31
    }
    track_script {
        chk_nginx
    }
    virtual_ipaddress {
        192.168.1.201/24
    }
}
```

#### 2.1.3 检测脚本


添加文件位置为/etc/keepalived/nginx_check.sh

```bash
#!/bin/bash
count=`ps -C nginx --no-header | wc -l`
if [ $count -eq 0 ]; then
    su - nginx lc "/usr/local/nginx/sbin/nginx"
    sleep 2
    if [ $count -eq 0 ]; then
        exit 1;
    fi
else
    exit 0;
fi
```

```bash
chmod 755 /etc/keepalived/nginx_check.sh
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