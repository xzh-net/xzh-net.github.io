# Kylin Server V10 SP3

- 官方网址：https://www.kylinos.cn

## 1. 安装

1. 选择图形化安装

![](../../assets/_images/deploy/kylin/1.png)

2. 选择语言

![](../../assets/_images/deploy/kylin/2.png)

3. 设置系统分区

![](../../assets/_images/deploy/kylin/3.png)

![](../../assets/_images/deploy/kylin/3_1.png)

4. 设置语言支持

![](../../assets/_images/deploy/kylin/4.png)

5. 设置软件选择

![](../../assets/_images/deploy/kylin/5.png)

![](../../assets/_images/deploy/kylin/5_1.png)

6. 设置网络和主机名

![](../../assets/_images/deploy/kylin/6.png)

![](../../assets/_images/deploy/kylin/6_1.png)

![](../../assets/_images/deploy/kylin/6_2.png)

7. 设置时间和日期

![](../../assets/_images/deploy/kylin/7.png)

![](../../assets/_images/deploy/kylin/7_1.png)

8. 设置root用户和密码

![](../../assets/_images/deploy/kylin/8.png)

![](../../assets/_images/deploy/kylin/8_1.png)

9. 点击开始安装

![](../../assets/_images/deploy/kylin/9.png)

10. 查看实时安装进度

![](../../assets/_images/deploy/kylin/10.png)

11. 安装完成，重启系统

![](../../assets/_images/deploy/kylin/11.png)

12. 同意许可协议，安装结束

![](../../assets/_images/deploy/kylin/12.png)

## 2. 虚拟机设置

### 2.1 初始化

```bash
nkvers  # 查看版本
yum install -y zip unzip telnet lsof ntpdate openssh-server wget net-tools.x86_64
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
/usr/sbin/ntpdate ntp4.aliyun.com;/sbin/hwclock -w      # 同步时间
systemctl stop firewalld.service
systemctl disable firewalld.service # 关闭
```

### 2.2 ssh配置

```bash
echo -e "ListenAddress 0.0.0.0\nPasswordAuthentication yes\nPermitRootLogin yes" >> /etc/ssh/sshd_config # 开启密码验证和root账号登录 

service sshd start
systemctl enable sshd   # 开机启动
```

### 2.3 网络配置

1. 设置静态ip

```bash
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=enp0s3
UUID=a49310b5-f18f-472b-b407-3ed5a385fa7c
DEVICE=enp0s3
ONBOOT=no
IPADDR=192.168.2.3
PREFIX=24
GATEWAY=192.168.2.1
DNS1=114.114.114.114
IPV6_PRIVACY=no
```

```bash
systemctl restart network
```

2. 设置dns

```bash
vim /etc/resolv.conf
```