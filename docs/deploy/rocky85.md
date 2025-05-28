# Rocky 8.5

- 官方网站：https://rockylinux.org

## 1. 系统安装

## 2. 初始化

### 2.1 配置静态IP地址

```bash
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
```

```apacheconf
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=enp0s3
UUID=c781471d-69e8-4363-95b1-0acbce583adb
GATEWAY=192.168.3.1
PREFIX=24
IPADDR=192.168.3.200
DNS1=223.6.6.6
DNS2=223.5.5.5
DNS3=114.114.114.114
DEVICE=enp0s3
ONBOOT=yes
```

```bash
systemctl restart NetworkManager    # 重启网络
ifdown enp0s3; ifup enp0s3
```

### 2.2 关闭防火墙

```bash
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
systemctl stop firewalld.service
systemctl disable firewalld.service # 关闭
systemctl enable --now cockpit.socket   # 开启web管理程序
systemctl disable cockpit.socket
```


### 3.3 更换软件源

```bash
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
    -i.bak \
    /etc/yum.repos.d/Rocky-*.repo
 
dnf makecache
```







